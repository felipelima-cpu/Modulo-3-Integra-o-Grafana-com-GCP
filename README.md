Bem-vindo ao laboratório de Rastreamento Distribuído! Este guia vai te ajudar a entender, configurar e analisar o comportamento de microserviços usando o Google Cloud Trace integrado com OpenTelemetry.

📖 1. O Problema: O "Mistério da Sexta-feira"
Imagine o seguinte cenário que provavelmente muitos já viveram: é sexta-feira à tarde e o monitoramento do banco dispara um alerta crítico: o tempo de resposta do serviço de transferências PIX subiu para 8 segundos.

Você corre para o Grafana e olha os painéis clássicos:

CPU das VMs? Normal.

Banco de dados? Normal.

Load Balancer? Normal.

O que está errado?
O problema era uma chamada a um microserviço de antifraude que, por sua vez, chamava um serviço de scoring externo. Essa cadeia de chamadas somava 6 segundos. Sem rastreamento distribuído, você não consegue ver isso — você só vê o sintoma de lentidão, não a causa raiz.

É exatamente para resolver esse problema que o Google Cloud Trace existe. Ele conecta os pontos entre todos os serviços pelos quais uma requisição passa, mostrando exatamente onde o tempo está sendo gasto.

🧠 2. Entendendo os Conceitos: Trace e Span
Pense no rastreamento como uma corrida de revezamento:

Trace (A Corrida): É a corrida inteira, do começo ao fim. Cada requisição do usuário gera um número único no placar, chamado de Trace ID.

Span (O Atleta): Cada atleta que carrega o bastão é um span. Ele tem um começo, um fim e passa o bastão para o próximo.

No nosso cenário bancário de PIX, um único Trace contém vários Spans: autenticação do usuário, validação de limite, consulta de saldo, atualização de extrato, etc. Se o bastão cair (ocorrer um erro), o Trace ID nos permite encontrar exatamente em qual atleta (span) o problema aconteceu, medido em milissegundos.

O Cloud Trace recebe esses dados de três formas:

Via SDK OpenTelemetry (Padrão recomendado e usado neste lab).

Via bibliotecas de cliente do GCP.

Via API REST direta.

⚙️ 3. Configurando o Ambiente
Antes de rodar o código, precisamos habilitar a API no Google Cloud e instalar as dependências do OpenTelemetry no Python.

Abra o seu terminal e execute:

Bash
# 1. Defina o ID do seu projeto (Substitua se necessário)
export PROJECT_ID="brb-monitoring-lab"

# 2. Habilite a API do Cloud Trace
gcloud services enable cloudtrace.googleapis.com --project=$PROJECT_ID

# 3. Confirme se a API está ativa
gcloud services list --enabled --filter="name:cloudtrace" --project=$PROJECT_ID

# 4. Instale as bibliotecas de telemetria do Python
pip3 install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-cloud-trace opentelemetry-exporter-otlp --break-system-packages
python3.11 -m pip install --upgrade google-cloud-trace protobuf opentelemetry-exporter-gcp-trace --break-system-packages

# 5. Verifique a instalação
pip list | grep opentelemetry
echo "✅ Dependências instaladas"
💻 4. O Código: Simulador de PIX
Crie um arquivo chamado trace_demo.py e cole o código abaixo. Esse script simula requisições de transferências PIX, passando por consulta de saldo e motor de antifraude, e envia a telemetria automaticamente para a nuvem.

Python
import time, random
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.trace.status import Status, StatusCode

# 1. Definir metadados do serviço
resource = Resource.create({
    SERVICE_NAME: "brb-transferencias",
    SERVICE_VERSION: "2.4.1",
    "deployment.environment": "lab",
    "team": "monitoramento-brb",
})

# 2. Configurar exporter para o GCP
exporter = CloudTraceSpanExporter(project_id="twitter-clone-sre")
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("brb.monitoramento")

# 3. Funções simuladas
def consultar_saldo(conta_id: str) -> float:
    with tracer.start_as_current_span("db.consultar_saldo") as span:
        span.set_attribute("db.system", "postgresql")
        span.set_attribute("db.name", "brb_core")
        span.set_attribute("db.operation", "SELECT")
        span.set_attribute("conta.id", conta_id)
        time.sleep(random.uniform(0.015, 0.06))
        span.set_status(Status(StatusCode.OK))
        return random.uniform(500.0, 80000.0)

def verificar_antifraude(valor: float, conta_id: str) -> bool:
    with tracer.start_as_current_span("fraude.verificar") as span:
        span.set_attribute("fraude.motor", "scoring-v3")
        span.set_attribute("transacao.valor", valor)
        time.sleep(random.uniform(0.08, 0.35)) # Gargalo simulado
        score = random.uniform(0.0, 1.0)
        aprovado = score < 0.85
        span.set_attribute("fraude.score", score)
        span.set_attribute("fraude.aprovado", aprovado)
        span.set_status(Status(StatusCode.OK))
        return aprovado

def processar_transferencia(de_conta: str, para_conta: str, valor: float):
    with tracer.start_as_current_span("transferencia.processar") as span:
        span.set_attribute("transacao.tipo", "PIX")
        span.set_attribute("transacao.de_conta", de_conta)
        span.set_attribute("transacao.para_conta", para_conta)
        span.set_attribute("transacao.valor", valor)

        try:
            saldo = consultar_saldo(de_conta)
            if valor > saldo:
                span.set_attribute("error.tipo", "saldo_insuficiente")
                raise ValueError(f"Saldo insuficiente: R$ {saldo:.2f}")

            aprovado = verificar_antifraude(valor, de_conta)
            if not aprovado:
                span.set_attribute("error.tipo", "bloqueado_antifraude")
                raise ValueError("Transação bloqueada pelo antifraude")

            time.sleep(random.uniform(0.02, 0.08))
            span.set_attribute("transacao.status", "CONCLUIDA")
            span.set_status(Status(StatusCode.OK))
            print(f"  ✅ PIX R$ {valor:.2f} de {de_conta} → {para_conta}")

        except ValueError as e:
            span.set_attribute("error", True)
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            raise e  

# 4. Execução
print("🚀 Iniciando simulações — os traces aparecerão no Cloud Trace\n")
for i in range(8):
    try:
        processar_transferencia(
            de_conta=f"BRB-{1000 + i:04d}",
            para_conta=f"BRB-{2000 + i:04d}",
            valor=random.uniform(50.0, 15000.0)
        )
    except ValueError as e:
        print(f"  ❌ Falha: {e}")
    time.sleep(0.5)

print("\n⏳ Aguardando flush dos spans...")
time.sleep(6)
print("✅ Verifique: console.cloud.google.com/traces")
Para rodar, basta executar: python3.11 trace_demo.py

📊 5. Interpretando os Resultados no Console
Após gerar os dados, abra a interface do Cloud Trace no Google Cloud Console. Aqui estão duas análises práticas que você conseguirá fazer com os dados gerados:

Cenário A: Análise de Desempenho Agregado (A Visão Macro)
Ao analisar o painel geral de traces do serviço, notamos comportamentos interessantes:

O fluxo principal transferencia.processar recebeu 56 requisições de transferência. O tempo mediano de resposta (P50) está aceitável, em torno de 257 milissegundos.

Porém, a taxa de erros é de 21,43%! Quase um quinto das transferências não foi concluído.

A etapa db.consultar_saldo rodou as mesmas 56 vezes. O banco de dados está super rápido (P50 de 34ms) e com zero erros.

A etapa fraude.verificar só rodou 50 vezes. O que isso nos diz? Que 6 transações falharam logo após o banco de dados (provavelmente barradas por saldo insuficiente), abortando a operação antes do antifraude.

Descobrimos também que o motor de antifraude é o nosso gargalo: consome em média 182ms, mas em cenários piores (P90/P95) passa dos 300ms, prejudicando o tempo total do PIX.

Cenário B: Raio-X de uma Falha Específica (A Visão Micro)
Quando clicamos em uma transação específica que falhou, o "gráfico em cascata" (Waterfall) nos conta a história detalhada:

No topo, a barra vermelha com exclamação indica que o fluxo total levou cerca de 328 milissegundos e estourou um erro.

Descendo as barras de tempo, vemos que o db.consultar_saldo (52ms) passou liso.

Logo depois, no fraude.verificar, a operação foi interrompida.

Ao abrir a aba "Logs & Events", o OpenTelemetry nos entrega o rastreamento completo da exceção (Stacktrace) capturada no código: "Transação bloqueada pelo antifraude" (ValueError na linha 80).
