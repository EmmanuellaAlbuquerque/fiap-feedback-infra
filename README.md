# FIAP Feedback Infra

Este repositório contém a infraestrutura compartilhada entre os microsserviços da Plataforma de Feedback, utilizando o AWS SAM (Serverless Application Model).

## 🚀 Tecnologias Utilizadas
*   **AWS SAM (Serverless Application Model)**: Para IaC (Infraestrutura como Código) e deploy.
*   **AWS CloudFormation**: A tecnologia subjacente usada pelo AWS SAM.

## ⚙️ Pré-requisitos
*   AWS CLI configurado com suas credenciais.
*   AWS SAM CLI instalado.

## 📦 Como Fazer o Deploy
**1. Valide o template SAM:**
```bash
sam validate --template-file template.yaml --region us-east-1
```

**2. Execute o deploy da stack:**
```bash
sam deploy --stack-name fiap-feedback-infra --region us-east-1
```

**3. Deletar template**
```
sam delete --stack-name fiap-feedback-infra
```
> **Importante:** O comando `deploy` fará o provisionamento ou atualização dos recursos na AWS de acordo com o `template.yaml`.

### Recursos Compartilhados entre os microsserviços

- MS1: https://github.com/MarcosBerto66/fiap_20252_servico_feedback_ingestao
- MS2: https://github.com/EmmanuellaAlbuquerque/fiap-feedback-notifier
- MS3: https://github.com/mainmtd/fiap-feedback-report-generator
- MS4: https://github.com/EmmanuellaAlbuquerque/fiap-feedback-admin

```mermaid
flowchart TB
    subgraph MS0["MS0: fiap-feedback-infra (Recursos Compartilhados)"]
        DB[("DynamoDB<br/>Tabela: FeedbackTable")]
        DB_Admins[("DynamoDB<br/>Tabela: AdminTable")]
        SQS_Urgency[("SQS<br/>Fila: FilaUrgencia")]
        S3_Reports[("S3<br/>Bucket: fiap-feedback-report-s3")]
        SNS_Reports[("SNS<br/>Tópico: Relatorios")]
    end

    classDef db fill:#336699,stroke:#333,stroke-width:1px,color:#fff;
    classDef queue fill:#ff9900,stroke:#333,stroke-width:1px,color:#fff;
    classDef s3 fill:#1f77b4,stroke:#333,stroke-width:1px,color:#fff;
    classDef sns fill:#ff9900,stroke:#333,stroke-width:1px,color:#fff;

    class DB,DB_Admins db;
    class SQS_Urgency queue;
    class S3_Reports s3;
    class SNS_Reports sns;
```

### Diagrama completo de todos os microsserviços

```mermaid
flowchart TB

%% ==========================================

%% 0. ATORES

%% ==========================================

Student((Estudante))

Admin((Administrador))



%% ==========================================

%% 1. CAMADA DE ENTRADA (APIs)

%% ==========================================

subgraph Input_Layer [ ]

style Input_Layer fill:transparent,stroke:none



subgraph MS1 ["MS1: fiap-feedback-ingest"]

API_Ingest["API Gateway<br>POST /avaliacao"]

Lambda_Ingest["Lambda: Ingestão"]

API_Ingest --> Lambda_Ingest

end



subgraph MS4 ["MS4: fiap-feedback-admin"]

API_Admin["API Gateway<br>POST /admin"]

Lambda_Admin["Lambda: Gerenciar Admins"]

API_Admin --> Lambda_Admin

end

end



%% ==========================================

%% 2. CAMADA DE INFRAESTRUTURA (Persistência & Mensageria)

%% ==========================================

subgraph Infra ["fiap-feedback-infra (Shared Resources)"]

direction LR

%% Definindo a ordem visual da esq para dir

DDB_Admins[("DynamoDB<br>Tabela: Admins")]

DDB_Feedbacks[("DynamoDB<br>Tabela: Feedbacks")]

SQS_Urgency[("SQS: FilaUrgencia")]

S3_Reports[("S3 Bucket<br>Relatórios")]

SNS_Reports{"SNS: Tópico<br>Relatórios"}

end



%% ==========================================

%% 3. CAMADA DE PROCESSAMENTO (Workers)

%% ==========================================

subgraph Async_Layer [ ]

style Async_Layer fill:transparent,stroke:none



subgraph MS2 ["MS2: fiap-feedback-notifier"]

Lambda_Urgency_Notifier["Lambda: Urgency Worker Notifier"]

Lambda_Report_Notifier["Lambda: Report Worker Notifier"]

SES["Amazon SES<br>Envio de E-mail"]

Lambda_Urgency_Notifier --> SES

Lambda_Report_Notifier --> SES


end



subgraph MS3 ["MS3: fiap-feedback-report"]

Scheduler("EventBridge<br>Cron Semanal")

Lambda_Report["Lambda: Gerador Relatório"]

Scheduler --> Lambda_Report

end

end



%% ==========================================

%% CONEXÕES (Fios Lógicos)

%% ==========================================



%% Fluxo Admin (Cadastro)

Admin -->|"1. Cadastra"| API_Admin

Lambda_Admin -->|"2. Persiste"| DDB_Admins



%% Fluxo Estudante (Ingestão)

Student -->|"3. Envia"| API_Ingest

Lambda_Ingest -->|"4. Salva"| DDB_Feedbacks

Lambda_Ingest -.->|"5. Se Nota < 5"| SQS_Urgency



%% Fluxo Notifier (Consumo)

SQS_Urgency -->|"6. Trigger"| Lambda_Urgency_Notifier

Lambda_Urgency_Notifier -->|"7. Lê E-mails"| DDB_Admins

SES -.->|"8. Alerta Admin"| Admin



%% Fluxo Report (Batch)

Lambda_Report -->|"9. Lê Dados"| DDB_Feedbacks

Lambda_Report -->|"10. Salva PDF"| S3_Reports

Lambda_Report -->|"11. Notifica"| SNS_Reports



%% Fluxo SNS -> MS2 -> SES

SNS_Reports -->|"12. Trigger"| Lambda_Report_Notifier

Lambda_Report_Notifier -->|"13. Lê PDF (Anexo)"| S3_Reports

SES -.->|"14. E-mail c/ Anexo"| Admin



%% ==========================================

%% ESTILOS (Classes)

%% ==========================================

classDef compute fill:#f9f,stroke:#333,stroke-width:2px

classDef storage fill:#336699,stroke:#333,stroke-width:2px,color:white

classDef msg fill:#ff9900,stroke:#333,stroke-width:2px,color:white

classDef ext fill:#DD344C,stroke:#333,stroke-width:2px,color:white



class Lambda_Ingest,Lambda_Admin,Lambda_Urgency_Notifier,Lambda_Report_Notifier,Lambda_Report compute

class DDB_Feedbacks,DDB_Admins,S3_Reports storage

class SQS_Urgency,SNS_Reports msg

class SES ext
```

---
**Desenvolvido para o Tech Challenge da FIAP - Fase de Cloud Computing & Serverless.**

