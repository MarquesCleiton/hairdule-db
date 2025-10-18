# HairduleDB — DynamoDB Stack (HairduleTB)

Stack responsável pela **infraestrutura de banco de dados** do ecossistema **Hairdule**.  
Implementada com **AWS SAM** e provisionada via **CloudFormation**, essa stack cria a tabela principal `HairduleTB` usada pelos serviços **Auth**, **Authorizer** e **Business**.

---

## 🧱 Estrutura da Stack

**Recurso principal:**  
- `AWS::DynamoDB::Table` → `HairduleTB`

**Modo de cobrança:**  
- `PAY_PER_REQUEST` (on-demand, dentro do Free Tier)

**TTL (Time to Live):**  
- Campo `expiresAt` habilitado — usado para expirar sessões automaticamente.

**Região padrão:**  
- `us-east-1`

---

## 🧩 Modelo de Dados (Single-Table Design)

A tabela foi projetada para armazenar **usuários**, **negócios**, **sessões** e **papéis (roles)** em uma **única tabela** com diferentes tipos de itens, definidos pelos prefixos `pk` e `sk`.

| PK | SK | Tipo | Descrição |
|----|----|------|------------|
| `USER#<uuid>` | `PROFILE` | Usuário | Dados principais do usuário |
| `USER#<uuid>` | `SESSION#<uuid>` | Sessão | Refresh token + TTL (`expiresAt`) |
| `BUSINESS#<uuid>` | `PROFILE` | Negócio | Dados do negócio |
| `BUSINESS#<uuid>` | `USER#<uuid>` | Vínculo | Relação Usuário ↔ Negócio |
| `ROLE#<name>` | `PROFILE` | Role | Template de permissões |

---

## 🗃️ Chaves e Índices

### 🔑 Primary Key (PK + SK)

| Campo | Tipo | Uso |
|--------|------|------|
| `pk` | String (HASH) | Particiona os grupos de dados (User, Business, Role, etc.) |
| `sk` | String (RANGE) | Ordena os tipos de item dentro do mesmo grupo |

**Exemplo:**
```
pk = USER#123
sk = PROFILE
```

---

### ⚙️ Global Secondary Indexes (GSIs)

#### 🔹 GSI1 — Vínculo Usuário ↔ Negócios

| Campo | Tipo | Descrição |
|--------|------|-----------|
| `gsi1pk` | HASH | ID do usuário (`USER#<uuid>`) |
| `gsi1sk` | RANGE | ID do negócio (`BUSINESS#<uuid>`) |

Usado para **listar negócios vinculados a um usuário**.

---

#### 🔹 GSI_EMAIL — Login por E-mail

| Campo | Tipo | Descrição |
|--------|------|-----------|
| `email` | HASH | E-mail do usuário (lowercase) |

Permite buscar um usuário diretamente pelo e-mail no login.

---

## 🧮 TTL (Time to Live)

Campo: `expiresAt`  
Usado apenas para **sessões (refresh tokens)**.  
Itens cujo `expiresAt` é menor que o timestamp atual são **excluídos automaticamente** pelo DynamoDB.

---

## 🧰 Estrutura do Projeto

```
hairdule-db/
├─ template.yaml
├─ samconfig.toml
├─ README.md
└─ .gitignore
```

---

## 🚀 Deploy

### Primeiro deploy (guiado)
```bash
sam build
sam deploy --guided
```

### Próximos deploys
Após o primeiro, basta:
```bash
sam build
sam deploy
```

O arquivo `samconfig.toml` guarda as configurações (stack, região, permissões etc.).

---

## 🧪 Verificando a tabela

### Pelo Console
1. Vá em [AWS → DynamoDB → Tables](https://console.aws.amazon.com/dynamodb/home?#tables)
2. Localize **HairduleTB**
3. Verifique:
   - **Billing mode:** Pay per request  
   - **Indexes:** GSI1 e GSI_EMAIL  
   - **TTL:** ativo no atributo `expiresAt`

### Pela CLI
```bash
aws dynamodb list-tables
aws dynamodb describe-table --table-name HairduleTB
```

---

## 🧭 Como modificar a tabela

### ➕ Criar novo GSI (exemplo)

Edite o arquivo `template.yaml`:

```yaml
GlobalSecondaryIndexes:
  - IndexName: GSI1
    KeySchema:
      - { AttributeName: gsi1pk, KeyType: HASH }
      - { AttributeName: gsi1sk, KeyType: RANGE }
    Projection: { ProjectionType: ALL }

  - IndexName: GSI_EMAIL
    KeySchema:
      - { AttributeName: email, KeyType: HASH }
    Projection: { ProjectionType: ALL }

  # Novo índice de exemplo
  - IndexName: GSI_STATUS
    KeySchema:
      - { AttributeName: status, KeyType: HASH }
    Projection: { ProjectionType: ALL }
```

Depois:
```bash
sam build
sam deploy
```

📘 **Importante:**  
O DynamoDB **não permite remover índices existentes**, apenas adicionar novos.  
Remover um índice requer recriar a tabela (atenção em produção).

---

### 🧾 Atualizar outros atributos

Pode adicionar novos campos à definição `AttributeDefinitions:` se forem usados em índices.

Exemplo:
```yaml
- { AttributeName: status, AttributeType: S }
```

Depois faça o `sam build` e `sam deploy`.

---

## 🧩 Outputs disponíveis

| Key | Descrição | Exemplo |
|-----|------------|----------|
| `HairduleTableArn` | ARN completo da tabela DynamoDB | `arn:aws:dynamodb:us-east-1:123456789012:table/HairduleTB` |
| `HairduleTableName` | Nome da tabela | `HairduleTB` |

Esses valores podem ser **referenciados por outras stacks** (como Auth e Business).

---

## 🔐 Permissões necessárias (IAM)

A Lambda Auth, Authorizer e Business precisarão das seguintes ações:

```yaml
Action:
  - dynamodb:GetItem
  - dynamodb:PutItem
  - dynamodb:UpdateItem
  - dynamodb:Query
  - dynamodb:DeleteItem
Resource:
  - arn:aws:dynamodb:${AWS::Region}:${AWS::AccountId}:table/HairduleTB
```

---

## 🧹 Boas práticas

- Nunca salve senhas em texto puro (use bcrypt no Auth Lambda)
- Ative o TTL apenas em sessões (não em usuários)
- Use nomes de índices legíveis e descritivos
- Prefira prefixos explícitos (USER#, BUSINESS#, ROLE#)
- Teste alterações em um ambiente separado antes de aplicar em produção

---

## 📜 Histórico de versões

| Versão | Alterações |
|--------|-------------|
| 1.0.0 | Criação da tabela HairduleTB com GSI1, GSI_EMAIL e TTL |
| 1.1.0 | Adição de documentação e suporte para novos GSIs via SAM |

---

## 🧠 Referências
- [AWS DynamoDB Docs](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
- [AWS SAM Reference](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-table.html)
- [AWS CLI DynamoDB Commands](https://docs.aws.amazon.com/cli/latest/reference/dynamodb/index.html)
