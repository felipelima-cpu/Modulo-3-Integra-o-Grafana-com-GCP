# Lab: OpenTelemetry + Google Cloud Trace

Guia rápido de execução do laboratório de rastreamento distribuído.

## 1. Configurar o Projeto e API
No seu Cloud Shell, defina seu projeto e habilite a API do Cloud Trace:

```bash
export PROJECT_ID="twitter-clone-sre"

gcloud services enable cloudtrace.googleapis.com --project=$PROJECT_ID
```

## 2. Instalar Dependências Python
Instale os pacotes do OpenTelemetry e os conectores do Google Cloud:

```bash
pip3 install opentelemetry-api opentelemetry-sdk opentelemetry-exporter-cloud-trace opentelemetry-exporter-otlp --break-system-packages

python3.11 -m pip install --upgrade google-cloud-trace protobuf opentelemetry-exporter-gcp-trace --break-system-packages
```

## 3. Criar o Script da Aplicação (`trace_demo.py`)
Crie o arquivo vazio:

```bash
nano trace_demo.py
```

Copie o bloco de código abaixo e cole dentro do editor. Todos os recuos (espaços) já estão corretos neste bloco. 

```python
import time, random
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.cloud_trace import CloudTraceSpanExporter
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.trace.status import Status, StatusCode

# 1. Configurar Provider e Exporter
resource = Resource.create({
    SERVICE_NAME: "brb-transferencias",
    SERVICE_VERSION: "2.4.1"
})

exporter = CloudTraceSpanExporter(project_id="twitter-clone-sre")
provider = TracerProvider(resource=resource)
provider.add_span_processor(BatchSpanProcessor(exporter))
trace.set_tracer_provider(provider)
tracer = trace.get_tracer("brb.monitoramento")

# 2. Funções Simuladas
def consultar_saldo(conta_id: str) -> float:
    with tracer.start_as_current_span("db.consultar_saldo") as span:
        span.set_attribute("db.system", "postgresql")
        time.sleep(random.uniform(0.015, 0.06))
        span.set_status(Status(StatusCode.OK))
        return random.uniform(500.0, 80000.0)

def verificar_antifraude(valor: float) -> bool:
    with tracer.start_as_current_span("fraude.verificar") as span:
        span.set_attribute("fraude.motor", "scoring-v3")
        time.sleep(random.uniform(0.08, 0.35)) 
        aprovado = random.uniform(0.0, 1.0) < 0.85
        span.set_status(Status(StatusCode.OK))
        return aprovado

def processar_transferencia(de_conta: str, para_conta: str, valor: float):
    with tracer.start_as_current_span("transferencia.processar") as span:
        try:
            saldo = consultar_saldo(de_conta)
            if valor > saldo:
                raise ValueError("Saldo insuficiente")

            if not verificar_antifraude(valor):
                raise ValueError("Bloqueado pelo antifraude")

            time.sleep(random.uniform(0.02, 0.08))
            span.set_status(Status(StatusCode.OK))
            print(f"  ✅ PIX R$ {valor:.2f} de {de_conta} → {para_conta}")

        except ValueError as e:
            span.record_exception(e)
            span.set_status(Status(StatusCode.ERROR, str(e)))
            raise e  

# 3. Execução
print("🚀 Enviando Traces...")
for i in range(8):
    try:
        processar_transferencia(f"BRB-{1000+i}", f"BRB-{2000+i}", random.uniform(50, 15000))
    except ValueError as e:
        print(f"  ❌ Falha: {e}")
    time.sleep(0.5)

time.sleep(6)
print("✅ Concluído!")
```

## 4. Executar e Visualizar
Rode o script para gerar os traces e enviá-los para a nuvem:

```bash
python3.11 trace_demo.py
```

**Resultado:** Acesse [console.cloud.google.com/traces](https://console.cloud.google.com/traces) no seu navegador para ver os gráficos de cascata gerados.
