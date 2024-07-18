---
title: "Caso de uso: LocalStack"
pubDate: 2024-06
description: "Utilizando LocalStack para simular o DynamoDB de forma local"
author: "José Rafael S. Hermoso"
---

# Caso de uso: LocalStack

## Um pouco sobre o LocalStack

O **LocalStack** é um emulador de serviços **AWS** que abrange seus principais serviços, alguns de forma gratuita e outros não. O objetivo dessa ferramenta é facilitar o desenvolvimento de aplicações que utilizam serviços da **AWS**, aumentando a segurança em relação a custos de desenvolvimento, melhorando a experiência do desenvolvedor em relação a problemas de configurações de permissões com o **IAM** e permitindo a existência de um ambiente de testes, visando tanto a parte de aprendizado e experimentação dos serviços da **AWS**, quanto processos como **CI** com integrações com o Github Actions.

O **LocalStack** também possui integrações com outras ferramentas incríveis como **Pulumi**, **Serverless**, **Terraform** e **Testcontainers**.

Leia mais sobre: [https://docs.localstack.cloud/getting-started/](https://docs.localstack.cloud/getting-started/)

## O cenário

Suponha uma aplicação simples que realiza todas as operações de um **CRUD** utilizando uma tabela no **DynamoDB**. Independentemente da forma com que o desenvolvedor vai construir essa aplicação, cedo ou tarde ele vai ter que acessar a tabela de produção para validar o que foi feito, seja com testes manuais que todos nós fazemos ou com **testes de integração** e **e2e**, e é aí que o problema começa a aparecer.

## O problema

O **DynamoDB** cobra por operações na tabela, ou seja, antes de realmente finalizar e disponibilizar a aplicação, já vão ter sido gerados custos. Adicione um pouco mais de complexidade nesse sistema, integrando um processamento assíncrono com **SQS**, eventos com **EventBridge** e notificações com **SNS** e **SES**, e pronto, sua fatura da AWS já vai estar rodando antes do dia 0 da sua aplicação.

## A solução

Nesse cenário, podemos utilizar em nosso ambiente de desenvolvimento o **LocalStack**, um emulador de serviços cloud **AWS** que tem como objetivo agilizar e simplificar o desenvolvimento e testes de aplicações que utilizem serviços da cloud **AWS**.

Utilizando o **Docker**, **docker-compose** e **AWS SDK** da linguagem de programação utilizada, conseguimos subir um container do **LocalStack** e, através da configuração de **URL** do **SDK** e de **variáveis de ambiente**, conseguimos manipular os ambientes para que, em desenvolvimento e testes, as chamadas apontem para o **LocalStack,** minimizando os custos durante o desenvolvimento.

No caso apresentado acima, conseguiríamos executá-lo totalmente dentro do **LocalStack** utilizando os serviços do **DynamoDB** e os serviços extras como **SQS**, **EventBridge**, **SNS** e **SES** (de forma simulada). Isso garantiria um **custo zero** de serviços cloud durante o desenvolvimento e em **pipelines** de **CI/CD**, uma vez que o **LocalStack** também possui integração com **GitHub Actions**.

## Implementação

Afim de exemplificar o uso e, embasado na aplicação apresentada acima, implementei parcialmente um **CRUD** simples de produtos com apenas duas operações: criar um item e ler todos os itens da tabela.

A implementação foi feita utilizando apenas recursos do Node 22, inclusive seu próprio test runner. A aplicação é uma API normal com duas rotas: uma para criar um item e outra para ler todos os itens de uma tabela do **DynamoDB**.

Para configurar meu ambiente de desenvolvimento, utilizei um arquivo **.env** para guardar minhas variáveis de acesso e **endpoint** da **AWS**. Esse arquivo será utilizado para que possamos alterar de forma rápida, sem ter que de fato abrir o código, o ambiente em que nossa aplicação vai rodar.

```tsx
NODE_ENV="dev"
PORT=3000
AWS_ENDPOINT=http://localhost:4566
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=fake_id
AWS_SECRET_ACCESS_KEY=fake_secret
ITEMS_TABLE_NAME="items_table"
```

**Também foi utilizado o LocalStack com Docker e docker-compose.**

```tsx
services:
  localstack:
    container_name: "localstack"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"            # LocalStack Gateway
      - "127.0.0.1:4510-4559:4510-4559"  # external services port range
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock" #required for some services
      - ./setup.sh:/etc/localstack/init/ready.d/start-localstack.sh

```

**No docker-compose, é importante ressaltar o último volume utilizado. Ele é um script .sh que será copiado para dentro do container do LocalStack e será executado junto com a inicialização do container. Esse arquivo contém o comando para criar uma tabela no DynamoDB.**

```tsx
#!/bin/bash

awslocal dynamodb create-table     --table-name items_table     --key-schema AttributeName=id,KeyType=HASH     --attribute-definitions AttributeName=id,AttributeType=S     --billing-mode PAY_PER_REQUEST     --region us-east-1

```

Todos os comandos para lidar com os serviços da **AWS** no **LocalStack** podem ser encontrados na documentação da ferramenta.

Com o container rodando e o arquivo **.env** configurado, o próximo passo é configurar via código o cliente do serviço que será utilizado. Nesse caso, o serviço será o **DynamoDBClient**.

**Vale ressaltar que foi utilizado o SDK v3 para o NodeJs.**

O **DynamoDBClient** recebe como parâmetro as seguintes configurações:

```jsx
const awsConfig = {
  endpoint: process.env.AWS_ENDPOINT,
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
};
```

Como passamos toda a nossa configuração via variáveis de ambiente, não precisamos alterar nenhuma parte do código para trocar entre nosso ambiente de desenvolvimento e o ambiente de produção.

A configuração do cliente do **DynamoDB** da nossa aplicação ficou da seguinte forma:

```jsx
import {
  CreateTableCommand,
  DeleteTableCommand,
  DynamoDBClient,
  PutItemCommand,
  ScanCommand,
} from "@aws-sdk/client-dynamodb";
import { PutCommand } from "@aws-sdk/lib-dynamodb";

const awsConfig = {
  endpoint: process.env.AWS_ENDPOINT,
  region: process.env.AWS_REGION,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  },
};

const dynamoClient = new DynamoDBClient(awsConfig);
export const Dynamo = {
  getAllItems: (tableName) => {
    return dynamoClient.send(
      new ScanCommand({
        TableName: tableName,
      })
    );
  },

  createItem: (item, tableName) => {
    return dynamoClient.send(
      new PutCommand({
        TableName: tableName,
        Item: {
          ...item,
        },
      })
    );
  },

  createTable: (tableName) => {
    return dynamoClient.send(
      new CreateTableCommand({
        TableName: tableName,
        KeySchema: [
          {
            AttributeName: "id",
            KeyType: "HASH",
          },
        ],
        AttributeDefinitions: [
          {
            AttributeName: "id",
            AttributeType: "S",
          },
        ],
        ProvisionedThroughput: {
          ReadCapacityUnits: 1,
          WriteCapacityUnits: 1,
        },
      })
    );
  },

  deleteTable: (tableName) => {
    return dynamoClient.send(
      new DeleteTableCommand({
        TableName: tableName,
      })
    );
  },
};
```

Como um dos intuitos do **LocalStack** é permitir o teste local de serviços **AWS**, criei uma rotina simples de teste, apenas para garantir que conseguimos executar as duas operações que a nossa aplicação se propõe a fazer.

```jsx
import { describe, it } from "node:test";
import { Dynamo } from "../dynamo-db.js";
import assert from "node:assert/strict";

describe("Integrations tests with DynamoDB and LocalStack", () => {
  const database = Dynamo;
  const testTableName = "items_table_test";
  it("it should create the items table", async () => {
    const response = await database.createTable(testTableName);
    const status = response["$metadata"].httpStatusCode;
    assert.equal(response != undefined, true);
    assert.equal(status, 200);
  });

  it("it should create a item", async () => {
    const response = await database.createItem(
      {
        id: "123",
        name: "teste",
        price: 100,
      },
      testTableName
    );

    const status = response["$metadata"].httpStatusCode;
    assert.equal(response != undefined, true);
    assert.equal(status, 200);
  });

  it("it should get all items", async () => {
    const response = await database.getAllItems(testTableName);
    const status = response["$metadata"].httpStatusCode;
    assert.equal(response != undefined, true);
    assert.equal(status, 200);
    assert.equal(response.Count > 0, true);
    assert.equal(response.ScannedCount > 0, true);
    assert.equal(response.Items.length > 0, true);
  });

  it("it should delete the items table", async () => {
    const response = await database.deleteTable(testTableName);
    assert.equal(response != undefined, true);
  });
});
```

**Ao final do desenvolvimento, criei uma action no GitHub para que possamos rodar nossos testes em um pipeline de CI/CD.**

```jsx
name: CI using localstack

on: push

jobs:
  continuos-integration:
    runs-on: ubuntu-latest
    environment: poc-node-js-localstack-env

    steps:
      - uses: actions/checkout@v3
      - name: Using Node.js
        uses: actions/setup-node@v2
        with:
          node-version: 22.

      - name: Start LocalStack
        uses: LocalStack/setup-localstack@v0.2.0
        with:
          image-tag: 'latest'
          install-awslocal: 'true'

      - name: Create .env file
        run: |
          touch .env
          echo "AWS_ACCESS_KEY_ID=${{vars.AWS_ACCESS_KEY_ID}}" >> .env
          echo "AWS_ENDPOINT=${{vars.AWS_ENDPOINT}}" >> .env
          echo "AWS_REGION=${{vars.AWS_REGION}}" >> .env
          echo "AWS_SECRET_ACCESS_KEY=${{vars.AWS_SECRET_ACCESS_KEY}}" >> .env
          echo "ITEMS_TABLE_NAME=${{vars.ITEMS_TABLE_NAME}}" >> .env
          cat .env

      - name: run install, build and test
        run: |
          npm install
          npm run test

```

Aqui está o link para mais informações sobre a integração do **LocalStack** com **GitHub Actions**: [https://docs.localstack.cloud/user-guide/ci/github-actions/](https://docs.localstack.cloud/user-guide/ci/github-actions/) .Esse recurso pode ajudar a configurar **pipelines** de **CI/CD** que utilizam o **LocalStack** para testes locais de serviços **AWS**.

## Limitações

Nem todos os serviços que podem ser emulados via **LocalStack** estão inteiramente implementados e estáveis. Pegando o **DynamoDB** e o **SES**, podemos notar que a maioria das funcionalidades do **DynamoDB** estão implementadas parcialmente e, para o SES, a maioria de seus serviços estão instáveis.

Com isso, podemos concluir que precisamos nos atentar aos serviços e suas funcionalidades para que não haja divergências bruscas entre nosso ambiente de desenvolvimento, testes e o de produção.

Na documentação do **LocalStack**, podemos encontrar todos os serviços e seus respectivos níveis de cobertura.

## Alguns exemplos

- docker-compose

```docker
services:
  localstack:
    container_name: "localstack"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"            # LocalStack Gateway
      - "127.0.0.1:4510-4559:4510-4559"  # external services port range
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock" #required for some services
```

- NodeJS DynamoDB example

```jsx
import { DynamoDBClient } from "@aws-sdk/client-dynamodb";

const dynamodbConfig = {
  region: "us-east-1",
};
const isLocal = IS_OFFLINE === "true";

if (isLocal) {
  const host = LOCALSTACK_HOST || "localhost";
  dynamodbConfig["endpoint"] = `http://${host}:4566`;
}

const client = new DynamoDBClient(dynamodbConfig);
```

- NodeJS SQS NestJS example

```tsx
@Injectable()
export class SqsService {
  private readonly client: SQSClient = new SQSClient({
    endpoint:
      this.envConfigService.getAwsEndpoint() || process.env.AWS_ENDPOINT,
    region: this.envConfigService.getAwsRegion(),
    credentials: {
      accessKeyId: this.envConfigService.getAwsAccessKeyId(),
      secretAccessKey: this.envConfigService.getAwsSecretAccessKey(),
    },
  });

  constructor() {}
}
```

```tsx
NODE_ENV=prod
AWS_ENDPOINT=protocol://service-code.region-code.amazonaws.com
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=real_id
AWS_SECRET_ACCESS_KEY=real_secret
```

```tsx
NODE_ENV=dev
AWS_ENDPOINT=http://localhost:4566
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=fake_id
AWS_SECRET_ACCESS_KEY=fake_secret
```

O **endpoint** é formado pelo seguinte padrão: “protocol://service-code.region-code.amazonaws.com”. Um exemplo de **endpoint** é “https://dynamodb.us-west-2.amazonaws.com”. Para o ambiente de desenvolvimento, o endpoint vai apontar para a porta que está rodando o container do LocalStack.

Vale ressaltar que as chaves e IDs de acesso podem ser simplesmente um “teste” para rodar de forma local.

## Recursos extras

- Serviços disponivéis: [localstack-services](https://docs.localstack.cloud/user-guide/aws/feature-coverage/)
- Lista de AWS SDKs: [AWS-SDKs](https://aws.amazon.com/developer/tools/)
- Docker: [Docker](https://docs.docker.com/engine/install/ubuntu/)
- LocalStack GitHub: [localstack-github](https://github.com/localstack/localstack)
- Aws Endpoints: [aws-endpoint-config](https://docs.aws.amazon.com/general/latest/gr/rande.html)
- Exemplo de uso: [localstack-test-erick-wendel](https://www.youtube.com/watch?v=rwyhw9UYHkA)
- Repo com código desenvolvido: [joserafaelSH/poc-node-js-localstack](https://github.com/joserafaelSH/poc-node-js-localstack)
