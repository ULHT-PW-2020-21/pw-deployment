# pw-deployment

outras referencias:
* https://django-environ.readthedocs.io/en/latest/
* https://stackoverflow.com/questions/50322966/changing-django-development-database-from-the-default-sqlite-to-postgresql
* django-heroku
* https://alicecampkin.medium.com/how-to-set-up-environment-variables-in-django-f3c4db78c55f

# Configurações para ambiente de produção 🏭
O ambiente de desenvolvimento é diferente do necessário para um site em produção. A sua configuração requer seguir uma [checklist](https://docs.djangoproject.com/en/3.2/howto/deployment/checklist/):

### Variáveis de ambiente

Vamos criar um ficheiro `.env` que guardará chaves e passwords assim como configurações específicas para ambiente desenvolvimento. Serão definidas como variáveis de ambiente em `.env`, que podem depois ser usadas noutros ficheiros. Para tal:
* na linha de comando instalar `pipenv install 'environs[django]==8.0.0'`  (eventualmente deverá precisar das plicas ')

* em `config/settings.py` adicionar no topo:
```python
# config/settings.py
from environs import Env 

env = Env()
env.read_env()

[...]
```
* crie um novo ficheiro chamado .env na mesma pasta que contém o manage.py. (é um ficheiro escondido (hidden file), não listado com ls, pois começa com '.').


### .gitignore
* Estamos a usar Git para controlo de fonte, e não queremos carregar tudo. Ignoremos alguns ficheiros.
* crie o ficheiro `.gitignore` com o seguinte conteúdo:
```
.env
__pycache__/
db.sqlite3
.DS_Store  # só para Mac
```


### Debug e ALLOWED HOSTS
* em desenvolvimento, é util o modo DEBUG, mas em desenvolvimento devemos alterar. Devemos tambem incluir o URL da aplicação Heroku como host. 

* em .env insira:
```
export DEBUG=True 
```

* Em config/settings.py insira:
```python
# config/settings.py
[...]

DEBUG = env.bool("DEBUG", default=False)
ALLOWED_HOSTS = ['.herokuapp.com', 'localhost', '127.0.0.1']
```
* isto permite que em desenvolvimento, DEBUG continue True, mas quando passarmos para produção, visto o .env não ser enviado para o Heroku (está no .gitignore), ficar False.


### Chave secreta

* Vamos mover o valor da variável SECRET_KEY de settings.py para .env, definindo-a como variável de ambiente da seguinte forma (sem as plicas '):
```
export DEBUG=True 
export SECRET_KEY=django-insecure-#nvkx1%+=m5nb9g^6a4k@!@&f@d@&v3!e7^#-1h8lo#)f9r9qy
```

* utilize a variável de ambiente em settings.py: 
```python
# config/settings.py

SECRET_KEY = env.str("SECRET_KEY")
```


### Base de dados PostgreSQL

* em ambiente de desenvolvimento local, usamos a base de dados SQLite. Mas em produção (no Heroku) devemos usar PostgreSQL, pois o ficheiro db.sqlite é apagado pelo Heroku.
* O módulo instalado environs[django] trata de todas as configurações necessárias
* atualize settings.py com:
```python
# config/settings.py

DATABASES = {
      "default": env.dj_db_url("DATABASE_URL") 
 }
```
* em .env especifique:
```
export DATABASE_URL=sqlite:///db.sqlite3
```
* Heroku cria uma base de dados nova PostgreSQL, e cria uma variavel de configuração chamada `DATABASE_URL`. Como .env não é carregado no Heroku, o nosso projeto Django usará no Heroku esta configuração PosrtgreSQL.
* com o ambiente virtual ativo, devemos instalar o adaptador de base de dados Psycopg, que põe o Python  a comunicar com bases de dados PostgreSQL.
```
> pipenv shell
> pipenv install psycopg2-binary==2.8.6
```


### Configurar ficheiros estaticos
Devemos configurar ficheiros estáticos, instalanr `whitenooise` e correr `collectstatic`



### Instalar Gunicorn como servidor web de produção

### Criar o `Procfile`

### Criar novo projeto Heroku e fazer push do código para Heroku

### Correr `heroku ps:scale web=1` para arrancar um processo web dyno

### `Debug` colocado a `False`

