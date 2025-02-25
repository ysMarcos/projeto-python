# Coleta de Dados da API da Marvel

## Descrição
Este script tem como objetivo realizar requisições à API da Marvel para coletar dados sobre personagens, quadrinhos e eventos, armazenando as informações em arquivos CSV e em um banco de dados SQLite.

## Requisitos
Antes de executar o script, certifique-se de que todas as bibliotecas necessárias estão instaladas. Utilize o seguinte comando para instalar as dependências:

```bash
pip install requests pandas sqlite3
```

O script também requer que as chaves de acesso da API da Marvel (publicKey e privateKey) estejam armazenadas no Google Colab através do `userdata`.  
As chaves podem ser acessadas ao realizar login ao Marvel Developer Portal com as credenciais de acesso `https://developer.marvel.com/`.  
![image](https://github.com/user-attachments/assets/121892ff-e3ae-48c4-b807-04182272c75e)


## Estrutura do Código
### 1. Importação de Bibliotecas que serão utilizadas
O script inicia importando as bibliotecas:
- `requests`: Para realizar requisições HTTP do tipo GET à API da Marvel.
- `hashlib`: Para gerar um hash MD5 necessário para autenticação na API conforme documentação Marvel.
- `pandas`: Para manipulação e estruturação dos dados coletados em um data frame, exportando em .CSV e armazenamento no banco.
- `sqlite3`: Para armazenar os dados em um banco de dados SQLite.
- `json`: Para manipulação de dados no formato JSON que retornam da requisição à API Marvel.
- `userdata` do Google Colab: Para armazenar e recuperar credenciais sensíveis.

### 2. Função de Geração do Hash MD5
A função `hashToMD5` gera um hash MD5 usando a chave privada, a chave pública e um timestamp, conforme exigido pela API da Marvel.  
O timestamp está setado manualmente como um str(1), mas pode ser modificado para algo dinâmico.

```python
def hashToMD5(publicKey, privateKey):
    return hashlib.md5(str.encode(str(1) + str(privateKey) + str(publicKey)))
```

### 3. Função de Requisição de Dados
A função `get` é responsável por fazer requisições à API da Marvel e retornar os dados no formato JSON que será manipulado posteriormente.

```python
def get(url: str, endpoint: str, headers: dict, params: dict):
    try:
        response = req.get(url=url+endpoint, headers=headers, params=params)
        return response.json()
    except Exception as e:
        print(e)
```

### 4. Armazenamento de Dados
#### Salvar em CSV
A função `saveToCSV` salva os dados do data frame em arquivos CSV utilizando `.to_csv` passando como parâmetro o nome e estrutura do arquivo.

```python
def saveToCSV(obj, csvPath):
    df = pd.DataFrame(obj)
    df.to_csv(csvPath + '.csv', encoding='utf-8', index=False, header=True)
```

#### Salvar em SQLite
A função `saveToSqlite` salva os dados em um banco SQLite e remove duplicatas baseando-se em colunas únicas.  
O trabalho de verificação de duplicados está a cargo do banco. Pode gerar ganho de performance em grandes volumes de dados.
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

Headers e parâmetros padrões:  
O `limit` padrão da requisição caso não seja informado é 20, e o limite máximo definido é 100.

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
O script coleta dados de três endpoints da API da Marvel: `characters`, `comics` e `events`, podendo ser acrescentados endpoints conforme necessidade de análises.

#### Coleta de Personagens:  
Para o retorno de Comics e Events, está sendo manipulado o objeto de retorno para armazenamento em uma lista, utilizando `','.join` passando como separador uma vírula, em conjunto com um loop `for`. 

```python
endpoint = 'characters'
result = get(url, endpoint, headers, params)['data']['results']
characters = [{
    "id": x['id'],
    "name": x["name"],
    "description": x["description"],
    "comics": ','.join([y["name"] for y in x["comics"]["items"]]),
    "events": ','.join([y["name"] for y in x["events"]["items"]])
} for x in result]
saveToCSV(csvPath=endpoint, obj=characters)
saveToSqlite(characters, endpoint, dbPath, ['id'])
```

#### Coleta de Quadrinhos  
O objeto de retorno manipulado na coleta de quadrinhos é `dataType`, `dates`, `pricetype`, `prices`, `events` e `creators`

```python
endpoint = 'comics'
result = get(url, endpoint, headers, params)['data']['results']
comics = [{
    "id": x["id"],
    "digitalId": x["digitalId"],
    "title": x["title"],
    "pageCount": x["pageCount"],
    "issueNumber": x["issueNumber"],
    "dateType": ','.join([y["type"] for y in x["dates"]]),
    "dates": ','.join([y["date"] for y in x["dates"]]),
    "priceType": ','.join([y["type"] for y in x["prices"]]),
    "prices": ','.join([str(y["price"]) for y in x["prices"]]),
    "events": ','.join([y["name"] for y in x["events"]["items"]]),
    "creators": ','.join(x["creators"])
} for x in result]
saveToCSV(csvPath=endpoint, obj=comics)
saveToSqlite(comics, endpoint, dbPath, ["id"])
```

#### Coleta de Eventos  
O objeto de reotno manipulado na coleta de eventos é `comics` e `characters`.

```python
endpoint = "events"
result = get(url, endpoint, headers, params)["data"]["results"]
events = [{
    "id": x["id"],
    "title": x["title"],
    "description": x["description"],
    "start": x["start"],
    "end": x["end"],
    "comics": ','.join([y["name"] for y in x["comics"]["items"]]),
    "characters": ','.join([y["name"] for y in x["characters"]["items"]])
} for x in result]
saveToCSV(csvPath=endpoint, obj=events)
saveToSqlite(events, endpoint, dbPath, ["id"])
```

### 7. Insights Para Análises de Dados  

Os dados coletados por meio deste script python, fornece insights úteis para tópicos de análises de dados, considerando informações disponíveis no retorno dos endpoints analisados.   


#### Análise de dados em Events  

Com base nos dados coletados em Events, é possível extrair elementos para diversas análises, como a quantidade de Comics publicados em cada evento, chegando ao evento mais populoso em quantidade de publicações, número de vendas por evento, total faturado no evento, faturamento médio por personagem, faturamento médio por evento, quantidade total de páginas por evento, média de páginas publicadas em cada evento, dentre outras possibilidades.   


#### Análise de dados em Comics  

Com base nos dados analisados em Comics, é possível calcular métricas como a quantidade de vendas de um determinado quadrinho, qual o quadrinho que teve o maior número de edições, preço médio de quadrinhos, faturamento total de determinada edição de quadrinho, número total de páginas publicadas, quantidade média de páginas por quadrinhos, sendo possível analisar se o público tem preferência a determinada quantidade de páginas, podendo determinar o modelo de publicações de novos quadrinhos no futuro, dentre outras análises possíveis com base nos dados de vendas, preço e quantidade de páginas por quadrinho.  


#### Análise de dados em Characters  

Dentre as possibilidades disponíveis de análises de dados em personagens, pode-se realizar verificações de participações de personagens em quadrinhos, chegando ao personagem que mais tem partiicpações, maior faturamento, e maior quantidade de páginas publicadas, sendo possível validar métricas de faturamento médio por personagem, maior faturamento de personagem, dentre outras possibilidades.

