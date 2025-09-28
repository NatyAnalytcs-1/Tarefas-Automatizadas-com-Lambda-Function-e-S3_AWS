# Projeto: Processamento de Arquivos com S3, Lambda e DynamoDB

Este projeto demonstra a criação de um fluxo de trabalho serverless (sem servidor) na AWS para processar arquivos. O objetivo é fazer o upload de um arquivo (JSON ou CSV) para um bucket S3, que dispara uma função Lambda para processar o conteúdo e salvar os dados em uma tabela no DynamoDB. Uma segunda função Lambda expõe esses dados através de um API Gateway.

Todo o ambiente será simulado localmente usando o **LocalStack**, o que permite desenvolver e testar aplicações de nuvem sem incorrer em custos na AWS.

## Arquitetura do Projeto

O fluxo de trabalho segue os seguintes passos:

1.  **Upload:** Um usuário envia um arquivo (ex: `notas_fiscais.json`) para um bucket no S3.
2.  **Gatilho (Trigger):** O evento de upload no S3 aciona automaticamente uma função Lambda.
3.  **Processamento e Armazenamento:** A função Lambda (`ProcessarNotasFiscais`) lê o conteúdo do arquivo, processa os dados e os insere em uma tabela no DynamoDB (`NotasFiscais`).
4.  **Consulta via API:** Uma API REST, criada com o API Gateway, permite consultar os dados armazenados no DynamoDB através de endpoints HTTP (GET e POST), que são atendidos por outra função Lambda.

![Diagrama da Arquitetura](https://i.imgur.com/u5v012c.png )

---

## 1. Pré-requisitos e Instalação

Antes de começar, você precisa ter o Python, o Docker e o AWS CLI instalados.

### 1.1. Instalação do LocalStack

O LocalStack simulará os serviços da AWS em sua máquina. A forma mais simples de instalá-lo é via `pip`.

```
pip install localstack
```
Para mais opções de instalação, consulte a documentação oficial.

### 1.2. Iniciando o LocalStack
Com o Docker em execução, inicie o LocalStack:
```
localstack start -d
```
O -d executa o processo em segundo plano (detached mode). Verifique se os serviços estão ativos:
```
localstack status services
```
O terminal do LocalStack estará disponível em http://localhost:4566.

### 1.3. Configurando o AWS CLI para o Ambiente Local
Vamos configurar o AWS CLI para apontar para o nosso ambiente LocalStack. As credenciais podem ser qualquer valor, pois o LocalStack não as valida.
```
aws configure
```
AWS Access Key ID: test
AWS Secret Access Key: test
Default region name: us-east-1
Default output format: json
Importante: Para que os comandos aws se comuniquem com o LocalStack, eles devem incluir o parâmetro --endpoint-url=http://localhost:4566.

## 2. Criação dos Recursos na AWS (via LocalStack )
Agora, vamos criar cada um dos recursos necessários para o nosso projeto.
### 2.1. Criar a Tabela no DynamoDB
Crie a tabela NotasFiscais com id como chave primária.

````
aws dynamodb create-table \
    --table-name NotasFiscais \
    --attribute-definitions AttributeName=id,AttributeType=S \
    --key-schema AttributeName=id,KeyType=HASH \
    --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5 \
    --endpoint-url=http://localhost:4566
````

Verifique se a tabela foi criada:
````
aws dynamodb list-tables --endpoint-url=http://localhost:4566
````


### 2.2. Criar o Bucket no S3
Crie o bucket que receberá os arquivos.
````
aws s3api create-bucket \
    --bucket notas-fiscais-upload \
    --region us-east-1 \
    --endpoint-url=http://localhost:4566
````

### 2.3. Criar a Função Lambda
Primeiro, crie um arquivo lambda_function.py com o código que processará o arquivo e o requirements.txt com as dependências. Depois, compacte-o em lambda_function.zip.
Crie a função Lambda ProcessarNotasFiscais. Atenção: substitua o arn:aws:iam::000000000000:role/lambda-role por uma role válida se estivesse na AWS real. No LocalStack, podemos usar um valor padrão.
````
aws lambda create-function \
    --function-name ProcessarNotasFiscais \
    --runtime python3.9 \
    --role arn:aws:iam::000000000000:role/lambda-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://lambda_function.zip \
    --endpoint-url=http://localhost:4566
````

### 2.4. Configurar o Gatilho (Trigger ) do S3 para a Lambda
Crie um arquivo notification.json com a seguinte estrutura para conectar o S3 à Lambda:
````
JSON
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}
````
Agora, aplique essa configuração de notificação ao bucket:
````
aws s3api put-bucket-notification-configuration \
    --bucket notas-fiscais-upload \
    --notification-configuration file://notification.json \
    --endpoint-url=http://localhost:4566
````
## 3. Testando o Fluxo de Upload
Vamos testar a primeira parte do fluxo.

### 3.1. Gerar Dados de Teste
Crie um script Python (gerar_dados.py ) para gerar um arquivo notas_fiscais.json com dados fictícios.

### 3.2. Enviar Arquivo para o S3
Copie o arquivo gerado para o bucket S3. Este comando irá disparar a Lambda.
````
aws s3 cp notas_fiscais.json s3://notas-fiscais-upload/ \
    --endpoint-url=http://localhost:4566
````
### 3.3. Verificar os Dados no DynamoDB
Após o upload, a Lambda deve ter processado o arquivo e gravado os dados. Você pode usar o NoSQL Workbench for DynamoDB ou o próprio AWS CLI para consultar a tabela NotasFiscais e verificar se os registros foram inseridos.

## 4. Criando a API para Consulta dos Dados
A segunda parte do projeto é expor os dados via API.

### 4.1. Criar a API REST
````
aws apigateway create-rest-api \
    --name "NotasFiscaisAPI" \
    --endpoint-url=http://localhost:4566
````
Anote o id da API retornado no JSON.

### 4.2. Obter o ID do Recurso Raiz
Use o ID da API do passo anterior para obter o ID do recurso raiz (/ ).
````
# Substitua <API_ID> pelo ID da sua API
aws apigateway get-resources \
    --rest-api-id <API_ID> \
    --endpoint-url=http://localhost:4566
````
Anote o id do recurso raiz.

### 4.3. Criar o Recurso /notas
````
# Substitua <API_ID> e <ROOT_RESOURCE_ID> pelos IDs anotados
aws apigateway create-resource \
    --rest-api-id <API_ID> \
    --parent-id <ROOT_RESOURCE_ID> \
    --path-part "notas" \
    --endpoint-url=http://localhost:4566
````
Anote o id do novo recurso /notas.

### 4.4. Criar os Métodos GET e POST
Vamos adicionar os métodos HTTP ao recurso /notas.
````
# Substitua <API_ID> e <NOTAS_RESOURCE_ID>
# Método GET
aws apigateway put-method --rest-api-id <API_ID> --resource-id <NOTAS_RESOURCE_ID> --http-method GET --authorization-type "NONE" --endpoint-url=http://localhost:4566

# Método POST
aws apigateway put-method --rest-api-id <API_ID> --resource-id <NOTAS_RESOURCE_ID> --http-method POST --authorization-type "NONE" --endpoint-url=http://localhost:4566
````
### 4.5. Integrar os Métodos com a Função Lambda
Conecte os métodos GET e POST para que eles invoquem a função ProcessarNotasFiscais.
````
# Substitua <API_ID> e <NOTAS_RESOURCE_ID>
# Integração GET
aws apigateway put-integration --rest-api-id <API_ID> --resource-id <NOTAS_RESOURCE_ID> --http-method GET --type AWS_PROXY --integration-http-method POST --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" --endpoint-url=http://localhost:4566

# Integração POST
aws apigateway put-integration --rest-api-id <API_ID> --resource-id <NOTAS_RESOURCE_ID> --http-method POST --type AWS_PROXY --integration-http-method POST --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:000000000000:function:ProcessarNotasFiscais/invocations" --endpoint-url=http://localhost:4566
````
### 4.6. Fazer o Deploy da API
Para que a API fique acessível, precisamos publicá-la em um "stage" (estágio ), como dev.
````
# Substitua <API_ID>
aws apigateway create-deployment \
    --rest-api-id <API_ID> \
    --stage-name dev \
    --endpoint-url=http://localhost:4566
````
## 5. Testando a API
Sua API agora está pronta para ser testada. O endpoint terá o seguinte formato:
http://localhost:4566/restapis/<API_ID>/dev/_user_request_/notas
Testar o Método GET (Consultar Dados )
````
powershell

Invoke-RestMethod -Uri "http://localhost:4566/restapis/<API_ID>/dev/_user_request_/notas" -Method GET
````
Testar o Método POST (Inserir Dados )
````
powershell

Invoke-RestMethod -Uri "http://localhost:4566/restapis/<API_ID>/dev/_user_request_/notas" `
    -Method POST `
    -ContentType "application/json" `
    -Body '{"id": "NF-MANUAL-001", "cliente": "Cliente Teste", "valor": 150.75, "data_emissao": "20
````

### Com isso, se completa todo o ciclo do projeto, desde o upload do arquivo até a consulta e inserção de dados via API, tudo em um ambiente de desenvolvimento local.
