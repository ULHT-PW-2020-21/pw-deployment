# pw-deployment

outras referencias:
* https://django-environ.readthedocs.io/en/latest/
* https://stackoverflow.com/questions/50322966/changing-django-development-database-from-the-default-sqlite-to-postgresql
* django-heroku

# Configurações para ambiente de produção 🏭
O ambiente de desenvolvimento é diferente do necessário para um site em produção. A sua configuração requer seguir uma [checklist](https://docs.djangoproject.com/en/3.2/howto/deployment/checklist/):

### Variáveis de ambiente
* na linha de comando instalar `pipenv install 'environs[django]==8.0.0'`  (eventualmente deverá precisar das plicas ')
* em `config/settings.py` adicionar no topo do ficheiro:
```python
# config/settings.py
from environs import Env  # novo

env = Env()  # novo
env.read_env()  # novo
```
* crie um novo ficheiro chamado .env na mesma pasta que contém o manage.py. Qualquer ficheiro iniciado com ponto, como .env, é um ficheiro escondido (hidden file), não sendo listado com ls.


### .gitignore
* Estamos a usar Git para controlo de fonte, nas não queremos seguir (*track*) tudo. Ignoremos alguns ficheiros.
* crie o ficheiro `.gitignore` com o seguinte conteúdo:
```
.env
__pycache__/
db.sqlite3
.DS_Store  # só para Mac
```


### Debug e ALLOWED HOSTS
* em desenvolvimento, é util o modo DEBUG. Em desenvolvimento devemos alterar. Deve tambem indicar Heroku como host. Em config/settings.py insira :
```python
# config/settings.py
DEBUG = env.bool("DEBUG", default=False)

ALLOWED_HOSTS = ['.herokuapp.com', 'localhost', '127.0.0.1']
```
* em .env insira:
```
export DEBUG=True 
```
* isto permite que em desenvolvimento, continue True, mas quando passarmos para produção, visto o .env não ser enviado para o Heroku (está no .gitignore), ficar False.


### Guardar `SECRET_KEY` secreta
* A variável SECRET_KEY é definida em settings.py quando é criado um novo projeto. 
```python
# config/settings.py
# SECURITY WARNING: keep the secret key used in production secret!
SECRET_KEY = 'django-insecure-#nvkx1%+=m5nb9g^6a4k@!@&f@d@&v3!e7^#-1h8lo#)f9r9qy'

```

* Vamos removê-la e guardá-la no ficheiro .env
```
SECRET_KEY = env.str("SECRET_KEY")
```


* Passe-a como uma variável para settings. 
```python
# config/settings.py

SECRET_KEY = env.str("SECRET_KEY")
```
* e guardá-la no ficheiro

### Configurar ficheiros estaticos
Devemos configurar ficheiros estáticos, instalanr `whitenooise` e correr `collectstatic`



### Instalar Gunicorn como servidor web de produção

### Criar o `Procfile`

### Criar novo projeto Heroku e fazer push do código para Heroku

### Correr `heroku ps:scale web=1` para arrancar um processo web dyno

### `Debug` colocado a `False`


### Base de dados de produção, não SQLite
