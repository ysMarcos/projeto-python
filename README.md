
#### Objetivo
Este código realiza integração com a API da Marvel para obter dados sobre personagens, quadrinhos e eventos. Ele segue as etapas de autenticação, requisição, transformação e armazenamento dos dados em um banco SQLite e arquivos CSV.

---

### Importação de Bibliotecas
- **requests**: Usada para realizar requisições HTTP.
- **hashlib**: Utilizada para gerar o hash MD5 necessário na autenticação.
- **pandas**: Para manipulação e transformação de dados.
- **sqlite3**: Para conexão e manipulação do banco de dados SQLite.
- **json**: Para manipulação de dados JSON.
- **userdata**: Utilizada para obter credenciais do usuário no Google Colab.

---

### Funções

#### `hashToMD5(publicKey, privateKey)`
- **Descrição**: Gera um hash MD5 para autenticação na API da Marvel.
- **Parâmetros**:
  - `publicKey`: Chave pública da API.
  - `privateKey`: Chave privada da API.
- **Retorno**: Objeto MD5 gerado.
- **Observação**: Usa um timestamp fixo (1) conforme o exemplo da documentação da API.

#### `get(url, endpoint, headers, params)`
- **Descrição**: Realiza requisições GET à API.
- **Parâmetros**:
  - `url`: URL base da API.
  - `endpoint`: Endpoint específico da API.
  - `headers`: Cabeçalhos HTTP.
  - `params`: Parâmetros de consulta.
- **Retorno**: Resposta JSON da requisição.
- **Tratamento de Erros**: Exibe exceções em caso de falha na requisição.

#### `mapper(keys, values)`
- **Descrição**: Mapeia dados JSON para um formato estruturado.
- **Parâmetros**:
  - `keys`: Lista de chaves desejadas nos dados JSON.
  - `values`: Lista de objetos JSON a serem mapeados.
- **Retorno**: Lista de dicionários com os dados mapeados.
- **Observações**:
  - Converte listas e dicionários para strings usando `json.dumps`.
  - É um mapeador genérico com limitações em tipagem.

#### `saveToCSV(obj, csvPath)`
- **Descrição**: Salva dados em um arquivo CSV.
- **Parâmetros**:
  - `obj`: Lista ou DataFrame contendo os dados.
  - `csvPath`: Caminho e nome do arquivo CSV.
- **Retorno**: Nenhum.

#### `saveToSqlite(obj, tableName, dbPath='bd.db', uniqueColumns=None)`
- **Descrição**: Salva dados em um banco SQLite.
- **Parâmetros**:
  - `obj`: Dados a serem inseridos (lista ou DataFrame).
  - `tableName`: Nome da tabela no banco.
  - `dbPath`: Caminho do arquivo SQLite.
  - `uniqueColumns`: Colunas para garantir unicidade (evitar duplicatas).
- **Retorno**: Nenhum.
- **Observação**: Usa SQL para eliminar registros duplicados com base nas colunas de unicidade.
- **Tratamento de Erros**: Exibe mensagens de erro em caso de falha.

---

### Variáveis Globais
- **`url`**: URL base da API da Marvel.
- **`publicKey` e `privateKey`**: Credenciais do usuário obtidas via `userdata`.
- **`securityHash`**: Hash MD5 gerado pela função `hashToMD5`.
- **`dbPath`**: Caminho para o arquivo SQLite.
- **`headers`**: Cabeçalhos HTTP padrão.
- **`params`**: Parâmetros padrão para as requisições à API.

---

### Fluxo de Execução
1. **Requisição de Personagens**:
   - Endpoint: `characters`.
   - Chaves mapeadas: `id`, `name`.
   - Salva os dados em `characters.csv` e na tabela `characters` no SQLite.

2. **Requisição de Quadrinhos**:
   - Endpoint: `comics`.
   - Chaves mapeadas: `id`, `digitalId`, `title`, `issueNumber`, `dates`, `prices`, `resourceURI`.
   - Salva os dados em `comics.csv` e na tabela `comics` no SQLite.

3. **Requisição de Eventos**:
   - Endpoint: `events`.
   - Chaves mapeadas: `id`, `title`, `description`, `resourceURI`, `start`, `end`.
   - Salva os dados em `events.csv` e na tabela `events` no SQLite.

---

### Observações Importantes
- **Autenticação**:
  - A API exige timestamp, chave pública e hash MD5.
  - O timestamp fixo usado no exemplo (1) pode ser modificado para valores dinâmicos.

- **Limitações do Mapper**:
  - Gera apenas strings para tipos complexos (listas e dicionários).
  - Pode ser substituído por uma implementação mais robusta.

- **Melhorias Possíveis**:
  - Implementar classes para representar os dados de forma mais estruturada.
  - Tornar o timestamp dinâmico.
  - Adicionar logs mais detalhados para monitoramento de erros e execução.

---

### Referências
- Documentação oficial da [API Marvel](https://developer.marvel.com/).
