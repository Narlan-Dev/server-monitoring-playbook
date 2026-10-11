# GUIA DE IMPLANTAÇÃO — MONITORAMENTO DE SAÚDE DE SERVIDOR

> Playbook genérico, derivado de uma implementação real já validada em produção. Serve como referência para implantar o mesmo tipo de monitoramento em **qualquer servidor Linux** que hospede aplicações web — não é específico de nenhum projeto, domínio ou organização. Onde houver valor específico de ambiente (domínio, IP, nome de app, usuário do SO), ele aparece como `<PLACEHOLDER>`.

## 0. Filosofia deste guia

Antes dos passos técnicos, os princípios que guiaram a implementação original e que valem para qualquer servidor:

1. **Nunca presuma — valide com dado real.** Nome de métrica, nome de componente, formato de log, versão de software: tudo isso varia entre versões e instalações. Em praticamente toda etapa deste guia há um passo de verificação antes do próximo passo de configuração. Pular a verificação é a causa mais comum de retrabalho.
2. **Construa em camadas, não tudo de uma vez.** Ao montar um pipeline novo (ex. coleta → filtro → transformação → exportação), valide cada camada isoladamente (com um exporter de debug/log) antes de acrescentar a próxima. Um pipeline de 5 etapas que falha é muito mais difícil de depurar do que 5 pipelines de 1 etapa testados em sequência.
3. **Calibre threshold de alerta com baseline real, nunca com um número "redondo" chutado.** Meça o estado atual do recurso antes de decidir o que é "normal" vs. "anômalo".
4. **Teste o caminho de notificação de ponta a ponta antes de depender dele.** Um alerta que nunca foi realmente testado é uma suposição, não uma garantia.
5. **Componentes marcados como experimentais/beta/alpha podem ter bugs reais, não só limitações documentadas.** Quando o comportamento não faz sentido (ex. um contador que diminui), não force uma correção superficial — investigue se o componente é confiável para o uso pretendido, e tenha um plano B.
6. **Prefira a solução mais agnóstica de tecnologia quando o ambiente tiver stacks heterogêneas.** Se há múltiplas aplicações com linguagens/frameworks diferentes atrás do mesmo proxy reverso, instrumentar no proxy custa uma implementação; instrumentar em cada aplicação custa N implementações.

---

## 1. Arquitetura de referência

```
                    ┌──────────────────┐
   fora do server → │  Serviço externo  │ → HTTP(s) uptime + expiração de SSL
                    │  de uptime check  │    de cada site hospedado
                    └─────────┬─────────┘
                              │ webhook
                              ▼
   dentro do server:   ┌────────────┐        ┌──────────────────┐
   Collector de      → │  Backend   │        │   Canal de chat   │
   telemetria          │de observa- │ ─────► │   (alertas)       │
   (métricas + logs) → │  bilidade  │ webhook│                   │
                        └────────────┘        └──────────────────┘
   Process manager (apps) → rotação/compressão de log local
```

Três frentes independentes, que juntas cobrem a saúde do servidor sem exigir mudança de código nas aplicações hospedadas:

| Frente | O que cobre | Onde roda |
|---|---|---|
| **Uptime + SSL** | O site responde? O certificado está perto de vencer? | Fora do servidor (serviço de terceiros, visão de fora pra dentro) |
| **Recursos do host** | CPU, RAM, disco, rede, swap | Dentro do servidor, via um Collector de telemetria |
| **Tráfego HTTP (opcional/avançado)** | Quais rotas são mais usadas, quais geram mais erro | Dentro do servidor, interceptando no proxy reverso |
| **Logs de aplicação** | Logs não crescem sem limite até estourar o disco | Dentro do servidor, via rotação automática |

O documento usa como implementação de referência (já validada): **OpenTelemetry Collector** (coleta) + **um backend de observabilidade compatível com OTLP** (armazenamento/visualização/alerta) + **um serviço de uptime check externo** + **um webhook de chat** (notificação) + **o gerenciador de processos das aplicações** (rotação de log). Troque os nomes de produto conforme o que seu ambiente já usa — a lógica e a sequência de validação se mantêm.

---

## 2. Pré-requisitos

Antes de começar, tenha em mãos:

- Acesso SSH com sudo ao servidor.
- Conta no backend de observabilidade escolhido, com credenciais de ingestão (endpoint + token/API key).
- Um webhook de canal de chat (Discord, Slack, etc.) para receber alertas — de preferência dedicado a alertas de infraestrutura, separado de qualquer canal de log de aplicação já existente.
- Conta no serviço de uptime check externo escolhido.
- Lista de pessoas que devem ter acesso de leitura/edição ao backend de observabilidade (para restringir depois).
- As URLs públicas de cada site/aplicação hospedada no servidor.

---

## 3. Parte 1 — Observabilidade de recursos do host

### 3.1 Instalar o Collector

Resolva a versão mais recente automaticamente em vez de hardcodar (evita links quebrados e erro de digitação de versão):

```bash
cd /tmp
VERSION=$(curl -s https://api.github.com/repos/open-telemetry/opentelemetry-collector-releases/releases/latest | grep -Po '"tag_name": "v\K[^"]*')
curl --proto '=https' --tlsv1.2 -fOL \
  "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${VERSION}/otelcol-contrib_${VERSION}_linux_<ARQUITETURA>.deb"
sudo apt install -y "./otelcol-contrib_${VERSION}_linux_<ARQUITETURA>.deb"
```

`<ARQUITETURA>` é `amd64` ou `arm64` — confirme antes com `uname -m` (`x86_64` → `amd64`; `aarch64` → `arm64`).

> **Ignore qualquer sugestão de instrumentar as aplicações** (SDKs de linguagem, variáveis `OTEL_EXPORTER_OTLP_ENDPOINT` apontando pra um receiver local) que o assistente de onboarding do seu backend de observabilidade possa sugerir nesta etapa. Isso é para telemetria de aplicação (APM/tracing), um projeto separado — aqui o objetivo é só saúde do host.

### 3.2 Credenciais de exportação — nunca em texto puro na config principal

O pacote cria um arquivo de ambiente referenciado pela unit do systemd (confirme com `systemctl cat otelcol-contrib | grep -i EnvironmentFile`). Acrescente as credenciais ali, não na config YAML:

```bash
AUTH=$(echo -n "<ID_DA_INSTANCIA>:<API_KEY>" | base64 -w0)

sudo tee -a /etc/otelcol-contrib/otelcol-contrib.conf > /dev/null <<EOF
OTEL_EXPORTER_ENDPOINT="<endpoint_fornecido_pelo_backend>"
OTEL_EXPORTER_AUTH_HEADER="Basic $AUTH"
EOF

unset AUTH
sudo chown root:root /etc/otelcol-contrib/otelcol-contrib.conf
sudo chmod 600 /etc/otelcol-contrib/otelcol-contrib.conf
```

> **Erro comum nº 1 — escaping de heredoc.** Se usar `$VAR` dentro de um heredoc não citado (`<<EOF`, sem aspas), ele é expandido na hora de escrever o arquivo — correto. Se escrever `\$VAR` (barra invertida antes do `$`) por engano, o arquivo recebe o texto literal `$VAR`, não o valor — e a autenticação falha silenciosamente até você inspecionar o arquivo gerado. Depois de qualquer heredoc com variável, **confira o conteúdo real do arquivo** (`cat`) antes de assumir que deu certo.

> **Erro comum nº 2 — placeholder não substituído.** Ao copiar um comando com `<algo>`, é fácil rodar literalmente com o texto `<algo>` em vez do valor real — isso gera uma credencial inválida que também falha silenciosamente (erro de autenticação, não de sintaxe). Prefira, quando possível, montar o comando já com os valores reais em vez de deixar o placeholder pra substituição manual de última hora.

### 3.3 Configurar os coletores de métrica do host

Comece pelo essencial (CPU, memória, disco) e adicione os demais conforme a necessidade:

```yaml
receivers:
  host_metrics:
    collection_interval: 30s
    scrapers:
      cpu:
        metrics:
          system.cpu.utilization:
            enabled: true
      memory:
        metrics:
          system.memory.utilization:
            enabled: true
      load: {}
      filesystem:
        metrics:
          system.filesystem.utilization:
            enabled: true
        exclude_mount_points:
          match_type: regexp
          mount_points:
            - "^/(dev|proc|sys|run|snap|var/lib/docker).*"
        exclude_fs_types:
          match_type: strict
          fs_types: ["tmpfs", "devtmpfs", "squashfs", "overlay"]
      disk: {}

processors:
  batch: {}
  resource_detection:
    detectors: [system]
    system:
      hostname_sources: [os]

exporters:
  otlp_http:
    endpoint: "${env:OTEL_EXPORTER_ENDPOINT}"
    headers:
      authorization: "${env:OTEL_EXPORTER_AUTH_HEADER}"

service:
  pipelines:
    metrics:
      receivers: [host_metrics]
      processors: [resource_detection, batch]
      exporters: [otlp_http]
```

**Pontos de atenção, validados na prática:**

- **Métricas "de utilização" (percentual 0–1) geralmente vêm desabilitadas por padrão.** Os scrapers de `cpu`/`memory`/`filesystem`/`paging` costumam expor só métricas brutas (tempo cumulativo de CPU, bytes de memória) a menos que você habilite explicitamente a métrica `*.utilization` correspondente, como no exemplo acima. Sem isso, você teria que calcular percentual a partir de contadores brutos na query — mais trabalho e mais chance de erro.
- **Exclua filesystems/dispositivos virtuais do scraper de `filesystem`**, senão loop devices (comuns em sistemas com Snap/Flatpak), `tmpfs`, `overlay` etc. entram no cálculo de uso de disco e distorcem o percentual real — o que quebra a calibração de qualquer alerta de disco baseado nessa métrica.
- **O scraper de `disk` tem seu próprio filtro, independente do de `filesystem`.** Ele identifica por dispositivo de bloco (`sda`, `dm-0`, `loop0`...), não por ponto de montagem — os mesmos loop devices vão aparecer aqui de novo e precisam de exclusão própria (`device!~"loop.*"` na query, já que o receiver não tem um filtro de exclusão nativo para esse scraper em todas as versões).
- **Rede**: exclua a interface de loopback (`lo`) das queries de tráfego — ela sempre reflete tráfego interno da própria máquina, simétrico entre entrada e saída, e não representa tráfego de rede real.
- **Processos específicos (ex. banco de dados, runtime de aplicação) exigem um scraper de `process` à parte**, com duas pegadinhas:
  - Sem filtro, ele cria uma série de métrica **por processo do sistema inteiro** (systemd, sshd, cron, etc.) — sempre restrinja com `include`/`exclude` por nome, listando só os processos que importam.
  - O Collector normalmente roda como um usuário de sistema sem privilégio — para ler métricas de processos de **outros** usuários (ex. o banco de dados roda como outro usuário do SO), é necessário conceder uma capability específica ao binário: `sudo setcap cap_sys_ptrace,cap_dac_read_search+ep /usr/bin/otelcol-contrib`. **Isso precisa ser reaplicado toda vez que o pacote for atualizado** (a atualização substitui o binário e zera a capability).
- **Nomes de componente têm aliases antigos e novos** (ex. `hostmetrics` vs. `host_metrics`) — a versão mais nova costuma funcionar com o nome antigo mas emite aviso de "deprecated"; prefira o nome atual.
- **Nem toda chave de métrica documentada existe em toda versão.** Antes de confiar numa chave de configuração obscura (ex. uma métrica opcional recém-adicionada à especificação), valide com o comando de validação da própria ferramenta (seção 3.4) — um erro de "chave inválida" é melhor descoberto ali do que em produção.

### 3.4 Subir o serviço e validar antes de confiar

**Sempre valide a sintaxe com as variáveis de ambiente carregadas antes de reiniciar o serviço** — rodar o validador sem carregar o arquivo de ambiente (`.conf`) vai reclamar de "endpoint não especificado" mesmo com a config correta, porque ele não tem como ler um arquivo `600 root:root` sem ser root:

```bash
sudo bash -c 'set -a; source /etc/otelcol-contrib/otelcol-contrib.conf; set +a; /usr/bin/otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml'
echo "saiu com código: $?"
```

Código `0` e nenhuma saída = válido. Só então:

```bash
sudo systemctl enable --now otelcol-contrib
sudo systemctl status otelcol-contrib
sudo journalctl -u otelcol-contrib --since "2 minutes ago" -o cat --no-pager | grep -iE "warn|error"
```

Erro comum a essa altura: `401`/`403` na exportação — quase sempre é o problema de escaping/placeholder da seção 3.2, não um problema de rede. Confira o arquivo de credenciais antes de suspeitar de outra coisa.

### 3.5 Confirmar visibilidade no backend de observabilidade

Não assuma o nome final da métrica — o gateway de ingestão do seu backend normaliza nomes OTel pra seu próprio padrão interno (ex. sufixos `_ratio`, `_total`, `_bytes`). Use a ferramenta de exploração ad-hoc do backend (ex. "Explore") pra:
1. Confirmar que as métricas aparecem, com os nomes e labels reais (não os nomes da especificação OTel).
2. Comparar o valor reportado com o que você vê localmente no servidor (`df -h`, `free -h`, `top`) — se não bater, a causa mais provável é um filtro de exclusão mal calibrado (seção 3.3).
3. Montar um dashboard mínimo com os painéis essenciais (CPU, RAM, disco), usando os nomes **confirmados**, não os previstos.

---

## 4. Parte 2 — Alertas de limiar

### 4.1 Calibrar com o baseline real

Antes de escolher um número de threshold, meça o estado atual real do recurso (via o dashboard da seção 3.5, ou direto no servidor). Um threshold "redondo" sem relação ao baseline real do ambiente (ex. 90% de disco num servidor que já roda cronicamente em 85%) não serve pra nada — ou nunca dispara, ou dispara toda hora.

### 4.2 Regra de alerta — dois níveis

Padrão recomendado: duas regras (warning/critical) sobre a mesma métrica, ou uma única regra com rotas diferentes por severidade, dependendo do que a ferramenta de alerta suportar:

- **Warning**: limiar mais baixo, "comece a prestar atenção".
- **Critical**: limiar mais alto, "aja agora".
- **Período de confirmação ("for"/"pending period")**: alguns minutos, pra não disparar por um pico transitório de 1 ciclo de avaliação.

Se o recurso monitorado puder legitimamente ficar sem nenhum dado no período (comum em contagens de evento raro, como erros — ver seção 7.7), configure explicitamente o comportamento de **"sem dado"** da regra: por padrão, muitas ferramentas de alerta tratam "ausência de dado" como "disparar", o que gera **falso positivo permanente** quando "sem dado" na verdade significa "tudo bem, nada aconteceu". Ajuste esse comportamento para "Normal"/"OK" quando for o caso.

Configure também o período de **"continuar disparado" pós-condição** (ex. "keep firing for"), se disponível — evita alternar "disparado"/"resolvido" repetidamente no canal de notificação quando a condição oscila perto do limiar.

### 4.3 Canal de notificação

1. Crie um contact point apontando para o webhook do canal de chat escolhido.
2. Teste a entrega pelo botão de teste nativo da ferramenta, se existir — isso confirma só a entrega, não o fluxo de regra completo (ver 4.4).
3. Associe a notification policy (a política default, se só existir esse tipo de alerta; uma política com rota/matcher dedicado, se for preciso separar por severidade/time).

### 4.4 Teste de ponta a ponta — não pule esta etapa

Um alerta nunca testado é uma suposição. Duas formas válidas de testar:

- **Validação orgânica**: se o recurso realmente cruzar o threshold durante a implementação (aconteceu, por exemplo, com disco durante uma limpeza/rotação de log), isso já serve como teste real — confirme que a notificação de disparo **e** a de resolução chegaram, com os valores das anotações corretamente preenchidos (anotações com variável de template só são preenchidas quando a regra está de fato em estado "Firing" — no editor da regra, antes de disparar, elas aparecem como texto cru do template, isso é esperado, não um bug).
- **Teste manual controlado**: baixe temporariamente o threshold para um valor abaixo do estado atual, force o disparo, confirme a notificação, **restaure o valor real** e confirme a notificação de resolução também.

---

## 5. Parte 3 — Monitoramento externo (uptime + SSL)

### 5.1 Uptime HTTP(s)

Para cada site/aplicação pública, um monitor HTTP(s) no serviço de uptime check externo escolhido, verificando de fora do servidor (importante: isso detecta problemas que o monitoramento interno não detecta, como firewall mal configurado, DNS, ou o próprio servidor inacessível).

### 5.2 Expiração de certificado SSL

Se o plano gratuito do serviço de uptime check não incluir isso (comum — funcionalidades de planos gratuitos mudam), alternativa sem custo: um script com cron no próprio servidor, usando `openssl` pra checar a validade de cada certificado e postar no mesmo canal de notificação se estiver perto de vencer:

```bash
#!/usr/bin/env bash
set -euo pipefail
source /etc/<diretorio_de_config>/webhook.env

DOMAINS=("<dominio1>" "<dominio2>" "<dominio3>")
ALERT_DAYS=" 30 14 7 3 1 0 "   # só alerta em marcos específicos, evita notificação repetida todo dia

send_notification() {
  local payload
  payload=$(jq -n --arg title "$1" --arg desc "$2" '{embeds: [{title: $title, description: $desc}]}')
  curl -s -H "Content-Type: application/json" -d "$payload" "$WEBHOOK_URL" > /dev/null
}

for domain in "${DOMAINS[@]}"; do
  expiry_date=$(echo | openssl s_client -servername "$domain" -connect "$domain:443" 2>/dev/null \
    | openssl x509 -noout -enddate 2>/dev/null | cut -d= -f2 || true)
  [ -z "$expiry_date" ] && { send_notification "SSL check falhou" "Não foi possível obter o certificado de $domain"; continue; }
  days_left=$(( ($(date -d "$expiry_date" +%s) - $(date +%s)) / 86400 ))
  if [ "$days_left" -lt 0 ] || [[ "$ALERT_DAYS" == *" $days_left "* ]]; then
    send_notification "Certificado SSL expirando" "$domain expira em $days_left dias ($expiry_date)"
  fi
done
```

Agende via `crontab -e` (ex. `0 8 * * * /caminho/do/script.sh`), guardando a URL do webhook num arquivo de ambiente `600 root:root` separado, nunca no próprio script.

---

## 6. Parte 4 — Gestão de log de aplicação

Logs de aplicação sem rotação são uma das causas mais comuns de disco cheio em servidores de produção. Se as aplicações são gerenciadas por um process manager (ex. PM2), use o módulo de rotação nativo dele; senão, `logrotate` do próprio sistema operacional.

```bash
<process_manager> install <modulo_de_logrotate>
<process_manager> set <modulo>:max_size 10M
<process_manager> set <modulo>:retain 14
<process_manager> set <modulo>:compress true
<process_manager> save
```

**Não assuma que a rotação/compressão automática funciona — valide com um arquivo real.** Depois de instalar:
1. Confirme que existe pelo menos um arquivo de log maior que o `max_size` configurado (se não, o teste não vai disparar nada).
2. Aguarde um ciclo do worker interno do módulo (geralmente configurável, segundos a minutos) e verifique se o arquivo foi renomeado/rotacionado.
3. **Confirme separadamente se a compressão (`.gz`) de fato ocorreu** — rotação e compressão podem ser etapas distintas internamente, e uma delas pode falhar silenciosamente (sem nenhuma linha de erro no log do próprio módulo) mesmo com a opção habilitada. Se a rotação funcionar mas a compressão não, o problema original (disco enchendo) não foi resolvido, só disfarçado — você vai acumular arquivos grandes não-compactados. Se isso acontecer, compacte manualmente (`gzip <arquivo>`) como mitigação imediata e trate como um bug a ser acompanhado, não como resolvido.

---

## 7. Parte 5 (avançada/opcional) — Observabilidade de tráfego HTTP via proxy reverso

Esta parte responde a uma pergunta diferente das anteriores: não "o servidor está de pé", mas **"o que está sendo usado, e onde estão os erros"**. É mais trabalhosa e tem mais pontos de falha — trate como incremento, não como parte obrigatória do monitoramento básico.

### 7.1 Por que instrumentar no proxy, não em cada aplicação

Se o servidor hospeda múltiplas aplicações com stacks de backend diferentes, um middleware de aplicação precisaria ser reimplementado em cada stack. O proxy reverso (ex. Nginx) na frente de todas elas é o único ponto verdadeiramente comum, independente de linguagem/framework — decisão de manter a solução agnóstica de tecnologia reaproveitando a mesma infraestrutura de coleta já instalada na Parte 1.

### 7.2 Log estruturado por site

Troque (ou confirme) o `log_format` do proxy para **JSON**, com um arquivo de log próprio por vhost/site (não um único arquivo global compartilhado — dificulta separar por site depois). Campos mínimos: timestamp, método HTTP, URI (sem query string), status, tempo de resposta, host.

```nginx
log_format access_json escape=json '{"time":"$time_iso8601","method":"$request_method","uri":"$uri","status":$status,"request_time":$request_time,"host":"$host"}';
```

Em cada vhost: `access_log /var/log/nginx/<site>-access.log access_json;`

> **Sempre valide a sintaxe antes de recarregar um proxy em produção** (`nginx -t` ou equivalente) — e edite um vhost por vez, validando entre cada edição, não todos de uma vez. É tráfego real de produção; um erro de sintaxe não detectado pode tirar todos os sites do ar na recarga.

### 7.3 Controle de cardinalidade — valide antes de implementar

"Rota mais utilizada" precisa significar o **padrão** da rota (ex. `/api/recursos/:id`), não a URL literal com um identificador embutido — senão cada identificador distinto (ID de banco, UUID) gera uma série de dados nova, e a cardinalidade explode.

**Antes de montar qualquer pipeline de coleta**, valide a estratégia de normalização contra uma amostra real do log de produção, com um script simples e descartável (qualquer linguagem de script serve):
1. Extraia método + caminho de cada linha do log real.
2. Aplique a normalização planejada (regex substituindo segmentos que parecem ID por um placeholder).
3. Conte quantos padrões de rota **distintos** sobram.

Se o número de padrões distintos continuar muito alto mesmo após a normalização, a causa mais provável não é a normalização em si, mas **tráfego que não é da aplicação**: bots/scanners testando centenas de caminhos aleatórios (ex. tentativas de login de plataformas populares, `/.env`, `/admin`), e rotas de front-end de single-page applications que não representam chamadas de API de negócio. A solução costuma ser mais simples que ajustar o regex: **filtrar por um prefixo que identifique de forma confiável as rotas de API de negócio** (ex. todo endpoint de API vive sob `/api/` ou `/backend/`) antes mesmo de normalizar — isso elimina a maior parte do ruído de uma vez.

### 7.4 Pipeline de coleta: ler, filtrar, normalizar

Reaproveitando o mesmo Collector da Parte 1, construído **em camadas, cada uma validada isoladamente** com um exporter de debug (que só grava o dado processado no próprio log do Collector, sem enviar a lugar nenhum) antes de acrescentar a próxima:

1. **Receiver de leitura de arquivo** (tail dos arquivos de log, parse de JSON por linha).
2. **Processor de filtro** (mantém só as linhas que batem com o prefixo de API de negócio decidido na seção 7.3).
3. **Processor de transformação** (normaliza o caminho: substitui segmentos no formato de ID — numérico, hexadecimal de tamanho fixo, UUID — por um placeholder).

Em cada camada, gere tráfego de teste real (uma requisição de cada tipo que você quer validar) e confira no log do próprio Collector se o resultado é o esperado — **não acumule as 3 camadas de uma vez só pra depois descobrir qual delas está errada.**

> **Cuidado ao testar com tráfego real concorrente**: um servidor de produção tem tráfego de usuários reais acontecendo o tempo todo. Uma janela de observação curta pode capturar tanto o seu teste quanto tráfego real simultâneo, tornando difícil isolar causa e efeito. Prefira testar com um caminho raro/exclusivo do teste, não um caminho popular da aplicação.

### 7.5 Métrica pré-agregada vs. log cru — uma lição aprendida

Existe a tentação de, dentro do próprio Collector, já transformar essas linhas de log filtradas/normalizadas numa métrica de contagem (um "connector" que soma ocorrências e gera um contador). **Teste esse comportamento com cuidado antes de confiar nele**: contadores precisam ter uma semântica de acumulação bem definida (total crescente desde o início, nunca decrescente) para serem compatíveis com a maioria dos backends de métricas. Se o componente usado para gerar a métrica estiver marcado como experimental/beta/alpha na documentação da ferramenta, valide explicitamente que o valor relatado **nunca diminui** ao longo de requisições repetidas de teste — um valor que oscila ou diminui é sinal de um problema real de implementação, não de configuração, e não deve ser "corrigido" só suprimindo o erro de validação do backend (isso mascara o sintoma sem corrigir a causa, entregando dado errado de forma silenciosa).

**Alternativa mais robusta, se o seu backend de observabilidade suportar armazenamento de log (ex. Loki e similares) além de métricas**: exporte as linhas já filtradas/normalizadas como **log estruturado** (não como métrica pré-computada), e faça a contagem **na hora da consulta** (ex. com uma função de contagem sobre o intervalo, no próprio dashboard). Isso elimina o problema de semântica de acumulação por completo — cada linha de log é um evento discreto, sem estado cumulativo envolvido.

**Pegadinha de sintaxe de consulta, específica desse caminho**: campos que chegam como metadado de cada linha de log (em vez de "label" indexado do fluxo de log inteiro) geralmente precisam ser filtrados **depois** do seletor de fluxo principal, não dentro dele — misturar os dois é um erro comum de sintaxe que resulta em "nenhum dado" sem mensagem de erro clara.

### 7.6 Dashboards de uso e erro

Com o dado (métrica ou log) disponível, dois painéis por site/aplicação:

- **Top N rotas mais usadas**: ranking — calcule como instantâneo do período visível (não recalculado a cada ponto da linha do tempo, senão rotas diferentes podem entrar/sair do ranking em pontos de tempo distintos e o painel mostra mais itens que o N configurado).
- **Taxa de erro por rota**: mesma lógica, filtrando por status de erro.

Para **taxa percentual de erro ao longo do tempo**, use uma **janela de agregação fixa** (ex. últimos 10 minutos), não uma janela que varia com o zoom do dashboard — com pouco tráfego, uma janela curta pode ter 1-2 requisições no total, e se por coincidência todas forem erro (inclusive erro gerado pelo seu próprio teste manual), o painel mostra "100% de erro" de forma matematicamente correta mas estatisticamente sem sentido. Janela fixa maior dilui esse ruído de amostra pequena.

### 7.7 Alerta de erro — decisões de design

- **Alerte em erro de servidor (5xx), não em erro de cliente (4xx).** Erro de cliente (não autenticado, recurso não encontrado) é tráfego esperado do dia a dia de qualquer aplicação exposta à internet — inclusive gerado por bots/scanners. Usar isso como gatilho de alerta produz ruído constante. Erro de servidor é sempre sinal de problema real no backend.
- **Alerte por contagem absoluta num período, não por percentual.** Mesmo problema de amostra pequena da seção 7.6 — 1 erro em 1 requisição é "100%", mas pode ser irrelevante; 50 erros em 5 minutos é relevante independente do percentual.
- **Uma regra cobrindo todos os sites, agrupando por identificador do site**, em vez de uma regra por site — se a ferramenta de alerta suportar múltiplas instâncias de uma mesma regra (uma por valor distinto do campo de agrupamento), um site com problema dispara independente dos outros, sem precisar duplicar a definição da regra.
- Lembre do ajuste de "comportamento sem dado" da seção 4.2 — esse tipo de alerta (contagem de evento raro) frequentemente retorna "sem dado" no estado saudável (zero erros = nada pra contar), e por padrão muitas ferramentas tratam isso como "disparar", gerando alarme falso permanente até o ajuste ser feito.

---

## 8. Checklist genérico de implantação

- [ ] Collector de telemetria instalado, autenticação validada, sem porta nova exposta publicamente
- [ ] Métricas de CPU/RAM/disco confirmadas no backend de observabilidade (nome real, não o previsto)
- [ ] Dashboard mínimo com os 3 recursos essenciais
- [ ] Alertas de limiar calibrados com baseline real, com comportamento de "sem dado" revisado
- [ ] Canal de notificação testado de ponta a ponta (disparo **e** resolução)
- [ ] Acesso ao backend de observabilidade restrito a quem precisa
- [ ] Uptime HTTP(s) de cada site monitorado de fora do servidor
- [ ] Expiração de SSL monitorada (nativo ou script próprio)
- [ ] Notificação de uptime/SSL integrada ao mesmo canal de alerta
- [ ] Rotação de log de aplicação instalada, com rotação **e compressão** confirmadas separadamente
- [ ] (Opcional) Observabilidade de tráfego HTTP por rota, com cardinalidade validada antes de implementar
- [ ] (Opcional) Alerta de erro 5xx por site

---

## 9. Catálogo de erros comuns

| Sintoma | Causa provável | Como confirmar |
|---|---|---|
| Autenticação falha (401/403) na exportação, config parece certa | Variável de ambiente com texto literal `\$VAR` em vez do valor (escaping de heredoc), ou placeholder não substituído | `cat` no arquivo de credenciais — deve mostrar o valor real, não `$VAR` nem `<placeholder>` |
| "Chave de configuração inválida" ao validar | Nome de métrica/opção não existe nesta versão do software | Rodar o comando de listagem de componentes/versão da ferramenta antes de usar uma chave não comum |
| Métrica de processo sempre zero, sem identificação de qual processo | Processo pertence a outro usuário do SO e falta capability de leitura cross-user | Conceder a capability específica ao binário do Collector; reaplicar após cada atualização de pacote |
| Percentual de disco/CPU não bate com o que o SO mostra | Filesystems/dispositivos virtuais (loop devices, tmpfs) entrando no cálculo | Excluir explicitamente no scraper correspondente — disco e filesystem têm filtros independentes |
| Alerta mostra texto de template não substituído (`{{ variável }}`) no editor da regra | Comportamento esperado — anotação só é preenchida quando a regra está de fato em estado de disparo | Não é bug; confirmar renderização correta durante o teste de ponta a ponta real |
| Dashboard de "Top N" mostra mais itens que N | Consulta recalculada por ponto no tempo em vez de uma vez para o período todo | Mudar o tipo de consulta do painel para instantânea |
| Painel de percentual mostra 0% ou 100% de forma instável | Janela de agregação dinâmica demais, poucas amostras por ponto | Trocar para janela fixa e mais larga |
| Alerta de contagem de evento raro fica permanentemente disparado mesmo sem o evento acontecer | Comportamento padrão da ferramenta de alerta ao receber "sem dado" | Ajustar explicitamente o comportamento de "sem dado" da regra para "Normal" |
| Métrica de contador criada via pipeline de observabilidade mostra valores que diminuem ao longo do tempo | Componente gerador de métrica não está respeitando semântica de acumulação cumulativa (comum em componentes marcados experimentais) | Não confiar no componente para esse uso; considerar caminho alternativo (ex. log cru + contagem em tempo de consulta) |
| Serviço de uptime/SSL de terceiros não oferece mais de graça um recurso que era gratuito | Mudança de política de plano do fornecedor, fora do seu controle | Ter um plano B simples (script próprio) documentado, não depender de um único fornecedor pra tudo |
