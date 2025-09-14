# AWS Lambda e S3 - Automação de Tarefas

Este repositório documenta minha experiência prática com AWS Lambda e Amazon S3, implementando automação de tarefas através de funções serverless que respondem a eventos do ambiente de armazenamento.

## 📋 Índice

- [Visão Geral do Projeto](#visão-geral-do-projeto)
- [Arquitetura da Solução](#arquitetura-da-solução)
- [Implementação da Lambda Function](#implementação-da-lambda-function)
- [Configuração do S3](#configuração-do-s3)
- [Anotações e Insights](#anotações-e-insights)
- [Recursos Úteis](#recursos-úteis)

## 🎯 Visão Geral do Projeto

Este laboratório prático teve como objetivo criar um sistema automatizado onde funções Lambda são acionadas por eventos do Amazon S3, demonstrando o poder da computação serverless para processamento de dados em tempo real.

## 🏗️ Arquitetura da Solução

Implementei uma arquitetura onde:
Upload no S3 → Event Notification → Lambda Function → Processamento → Ações

```

**Fluxo principal:**
1. Arquivo é enviado para bucket S3
2. S3 dispara notificação para Lambda
3. Lambda executa função de processamento
4. Resultados são salvos em outro bucket ou banco de dados

## ⚙️ Implementação da Lambda Function

### Função Python para Processamento de Arquivos

```python
import json
import boto3
from datetime import datetime

s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('FileProcessingLogs')

def lambda_handler(event, context):
    try:
        # Obter informações do evento do S3
        bucket_name = event['Records'][0]['s3']['bucket']['name']
        file_key = event['Records'][0]['s3']['object']['key']
        
        # Log do processamento
        print(f"Processando arquivo: {file_key} do bucket: {bucket_name}")
        
        # Obter metadados do arquivo
        response = s3_client.head_object(Bucket=bucket_name, Key=file_key)
        file_size = response['ContentLength']
        last_modified = response['LastModified']
        
        # Processar conforme o tipo de arquivo
        if file_key.endswith('.csv'):
            process_csv(bucket_name, file_key)
        elif file_key.endswith('.json'):
            process_json(bucket_name, file_key)
        elif file_key.endswith('.txt'):
            process_text(bucket_name, file_key)
        
        # Registrar no DynamoDB
        log_processing(bucket_name, file_key, file_size, 'SUCCESS')
        
        return {
            'statusCode': 200,
            'body': json.dumps('Processamento concluído com sucesso!')
        }
        
    except Exception as e:
        print(f"Erro no processamento: {str(e)}")
        log_processing(bucket_name, file_key, file_size, 'ERROR', str(e))
        raise e

def process_csv(bucket_name, file_key):
    """Processa arquivos CSV"""
    print(f"Processando CSV: {file_key}")
    # Lógica específica para CSV
    
def process_json(bucket_name, file_key):
    """Processa arquivos JSON"""
    print(f"Processando JSON: {file_key}")
    # Lógica específica para JSON

def process_text(bucket_name, file_key):
    """Processa arquivos de texto"""
    print(f"Processando texto: {file_key}")
    # Lógica específica para texto

def log_processing(bucket_name, file_key, file_size, status, error_message=None):
    """Registra processamento no DynamoDB"""
    table.put_item(
        Item={
            'FileKey': file_key,
            'BucketName': bucket_name,
            'FileSize': file_size,
            'ProcessingTime': datetime.now().isoformat(),
            'Status': status,
            'ErrorMessage': error_message
        }
    )
```

## Configuração do IAM Role
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::input-bucket/*",
                "arn:aws:s3:::input-bucket"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::output-bucket/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:UpdateItem"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/FileProcessingLogs"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        }
    ]
}
```

## Configuração do S3

Notificação de Eventos do S3
```
{
    "LambdaFunctionConfigurations": [
        {
            "Id": "ObjectCreatedEvent",
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:process-s3-files",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {
                "Key": {
                    "FilterRules": [
                        {
                            "Name": "suffix",
                            "Value": ".csv"
                        }
                    ]
                }
            }
        },
        {
            "Id": "ObjectCreatedAllEvents",
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:process-s3-files",
            "Events": ["s3:ObjectCreated:*"]
        }
    ]
}
```
## Policy do Bucket S3 para Lambda
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::input-bucket/*"
        }
    ]
}
```

## Implementação com AWS CLI
Comandos para Configuração
```
# Criar função Lambda
aws lambda create-function \
    --function-name process-s3-files \
    --runtime python3.9 \
    --role arn:aws:iam::123456789012:role/lambda-s3-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip

# Configurar notificação do S3
aws s3api put-bucket-notification-configuration \
    --bucket input-bucket \
    --notification-configuration file://notification.json

# Testar a função Lambda
aws lambda invoke \
    --function-name process-s3-files \
    --payload file://test-event.json \
    output.txt
```

## Exemplo de Evento de Teste
```
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2023-01-01T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "s3SchemaVersion": "1.0",
        "configurationId": "testConfigRule",
        "bucket": {
          "name": "input-bucket",
          "ownerIdentity": {
            "principalId": "EXAMPLE"
          },
          "arn": "arn:aws:s3:::input-bucket"
        },
        "object": {
          "key": "test-file.csv",
          "size": 1024,
          "eTag": "0123456789abcdef0123456789abcdef",
          "versionId": "null"
        }
      }
    }
  ]
}
```

## Anotações e Insights

Lições Aprendidas:
Vantagens da Arquitetura Serverless: Custo-efetividade: Paga apenas pelo tempo de execução
Escalabilidade automática: Lida com picos de carga automaticamente
Baixa manutenção: Sem gerenciamento de servidores

Padrões Comuns de Integração:
S3 → Lambda: Processamento de arquivos em tempo real
Lambda → DynamoDB: Registro de logs e metadados
Lambda → S3: Armazenamento de resultados

Melhores Práticas Implementadas:
Tratamento robusto de erros
Logging detalhado para debugging
Permissões mínimas necessárias no IAM
Timeouts apropriados para diferentes cargas de trabalho

Otimizações de Performance:
Uso de versões compiladas do Python para melhor performance
Conexões persistentes com serviços AWS
Processamento em lote quando possível

Monitoramento e Troubleshooting:
CloudWatch Logs para monitoramento
Métricas customizadas para business metrics
Alertas para falhas de processamento


## Próximos Passos
Para aprofundar os conhecimentos em Lambda e S3, planejo:
Implementar processamento assíncrono com SQS
Explorar Lambda Layers para dependências compartilhadas
Implementar versionamento e aliases para deployments
Explorar processamento de grandes volumes de dados com Step Functions
Implementar autenticação e autorização com Cognito
