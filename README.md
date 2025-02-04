# Documentação do Script de Coleta de Dados da API da Marvel

## Descrição
Este script tem como objetivo realizar requisições à API da Marvel para coletar dados sobre personagens, quadrinhos e eventos, armazenando as informações em arquivos CSV e em um banco de dados SQLite.

## Requisitos
Antes de executar o script, certifique-se de que todas as bibliotecas necessárias estão instaladas. Utilize o seguinte comando para instalar as dependências:

```bash
pip install requests pandas sqlite3
```

O script também requer que as chaves de acesso da API da Marvel (publicKey e privateKey) estejam armazenadas no Google Colab através do `userdata`.

## Estrutura do Script
### 1. Importação de Bibliotecas
O script inicia importando as bibliotecas necessárias:
- `requests`: Para realizar requisições HTTP à API da Marvel.
- `hashlib`: Para gerar um hash MD5 necessário para autenticação na API.
- `pandas`: Para manipulação e estruturação dos dados coletados.
- `sqlite3`: Para armazenar os dados em um banco de dados SQLite.
- `json`: Para manipulação de dados no formato JSON.
- `userdata` do Google Colab: Para armazenar e recuperar credenciais sensíveis.

### 2. Função de Geração do Hash MD5
A função `hashToMD5` gera um hash MD5 usando a chave privada, a chave pública e um timestamp, conforme exigido pela API da Marvel.

```python
def hashToMD5(publicKey, privateKey):
    return hashlib.md5(str.encode(str(1) + str(privateKey) + str(publicKey)))
```

### 3. Função de Requisição de Dados
A função `get` é responsável por fazer requisições à API da Marvel e retornar os dados no formato JSON.

```python
def get(url: str, endpoint: str, headers: dict, params: dict):
    try:
        response = req.get(url=url+endpoint, headers=headers, params=params)
        return response.json()
    except Exception as e:
        print(e)
```

### 4. Funções de Armazenamento de Dados
#### Salvamento em CSV
A função `saveToCSV` salva os dados em arquivos CSV.

```python
def saveToCSV(obj, csvPath):
    df = pd.DataFrame(obj)
    df.to_csv(csvPath + '.csv', encoding='utf-8', index=False, header=True)
```

#### Salvamento em SQLite
A função `saveToSqlite` salva os dados em um banco SQLite e remove duplicatas baseando-se em colunas únicas.

```python
def saveToSqlite(obj, tableName, dbPath='bd.db', uniqueColumns=None):
    try:
        df = pd.DataFrame(obj) if not isinstance(obj, pd.DataFrame) else obj
        with sqlite3.connect(dbPath) as con:
            df.to_sql(tableName, con, if_exists='append', index=False)
            if uniqueColumns:
                uniqueClause = ", ".join(uniqueColumns)
                query = f"""
                    DELETE FROM {tableName}
                    WHERE rowid NOT IN (
                        SELECT MIN(rowid)
                        FROM {tableName}
                        GROUP BY {uniqueClause}
                    );
                """
                con.execute(query)
    except Exception as e:
        print(f"Erro: {e}")
```

### 5. Configuração de Variáveis
As credenciais e a URL base da API da Marvel são recuperadas a partir do `userdata`.

```python
url = 'http://gateway.marvel.com/v1/public/'
publicKey = userdata.get('publicKey')
privateKey = userdata.get('privateKey')
securityHash = hashToMD5(publicKey, privateKey).hexdigest()
dbPath = userdata.get('dbPath')
```

Headers e parâmetros padrões são definidos:

```python
headers = {'Accept': "*/*"}
params = {
    "ts": 1,
    "apikey": publicKey,
    "hash": securityHash,
    "limit": 100
}
```

### 6. Coleta de Dados e Armazenamento
O script coleta dados de três endpoints da API da Marvel: `characters`, `comics` e `events`.

#### Coleta de Personagens

```python
endpoint = 'characters'
result = get(url, endpoint, headers, params)['data']['results']
characters = [{
    "id": x['id'],
    "name": x["name"],
    "description": x["description"],
    "comics": ''.join([y["name"] for y in x["comics"]["items"]]),
    "events": ''.join([y["name"] for y in x["events"]["items"]])
} for x in result]
saveToCSV(csvPath=endpoint, obj=characters)
saveToSqlite(characters, endpoint, dbPath, ['id'])
```

#### Coleta de Quadrinhos

```python
endpoint = 'comics'
result = get(url, endpoint, headers, params)['data']['results']
comics = [{
    "id": x["id"],
    "digitalId": x["digitalId"],
    "title": x["title"],
    "pageCount": x["pageCount"],
    "issueNumber": x["issueNumber"],
    "dateType": ''.join([y["type"] for y in x["dates"]]),
    "dates": ''.join([y["date"] for y in x["dates"]]),
    "priceType": ''.join([y["type"] for y in x["prices"]]),
    "prices": ''.join([str(y["price"]) for y in x["prices"]]),
    "events": ''.join([y["name"] for y in x["events"]["items"]]),
    "creators": ''.join(x["creators"])
} for x in result]
saveToCSV(csvPath=endpoint, obj=comics)
saveToSqlite(comics, endpoint, dbPath, ["id"])
```

#### Coleta de Eventos

```python
endpoint = "events"
result = get(url, endpoint, headers, params)["data"]["results"]
events = [{
    "id": x["id"],
    "title": x["title"],
    "description": x["description"],
    "start": x["start"],
    "end": x["end"],
    "comics": ''.join([y["name"] for y in x["comics"]["items"]]),
    "characters": ''.join([y["name"] for y in x["characters"]["items"]])
} for x in result]
saveToCSV(csvPath=endpoint, obj=events)
saveToSqlite(events, endpoint, dbPath, ["id"])
```

