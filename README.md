# 10 passos para implantação duma aplicação Django num ambiente de produção no Heroku 🏭

* Para tornar uma aplicação visível na Internet temos que fazer implantação (*deployment*) do código num servidor e base de dados externa. Chama-se isto pôr o código em ambiente de produção.
* Várias coisas devem ser feitas. 
    * Várias configurações devem ser adaptadas caso estejamos em ambiente de produção
    * Devemos usar um serviço de Web hosting (usaremos Heroku)
    * O servidor Web do Django para uso local é básico, não serve para produção (usaremos gunicorn). 
    * A base de dados SQLite só serve em ambiente de desenvolvimento, devendo ser usada uma outra (usaremos PostgreSQL). .
    * Os ficheiros estáticos devem ser configurados para ambiente produção.
* A seguir detalham-se todos os passos necessários.

## 1. Variáveis de ambiente

Vamos criar um ficheiro `.env` que guardará chaves e passwords assim como configurações específicas para ambiente desenvolvimento. Serão definidas como variáveis de ambiente em `.env`, que podem depois ser usadas noutros ficheiros. Para tal:
* na linha de comando instalar environs (eventualmente deverá precisar das plicas ''):
```
> pipenv shell
> pipenv install 'environs[django]==8.0.0'
```

* em `config/settings.py` adicionar no topo:
```python
# config/settings.py
from environs import Env 

env = Env()
env.read_env()

[...]
```
* crie um novo ficheiro chamado `.env` na mesma pasta que contém o manage.py. (é um ficheiro escondido (hidden file), não listado com ls, pois começa com '.').


## 2. .gitignore
* crie o ficheiro `.gitignore` indicando os ficheiros a ser ignorados pelo GIT:
```
.env
__pycache__/
db.sqlite3
.DS_Store   # só para Mac
```
* o facto de não carregarmos `.env` para o Heroku permite que certas configurações só estejam ativas no ambiente de desenvolvimento.

## 3. Debug 
* É necessário especificar que em desenvolvimento o modo DEBUG fique ativo, mas em produção não. 
* em `.env` insira:
```
export DEBUG=True 
```

* Em `config/settings.py` altere o valor para DEBUG:
```python
# config/settings.py
[...]

DEBUG = env.bool("DEBUG", default=False)
```

## 4. ALLOWED HOSTS
* Em `settings.py`inclua o URL da aplicação Heroku como host em ALLOWED_HOSTS:
```python
# config/settings.py
[...]

ALLOWED_HOSTS = ['a-minha-app-heroku.herokuapp.com', 'localhost', '127.0.0.1']
```

## 5. SECRET_KEY

* Vamos mover o valor da variável SECRET_KEY (especifica do seu projeto) de settings.py para .env, definindo-a como variável de ambiente da seguinte forma (sem as plicas ').
* Em `.env` insira:
```
export SECRET_KEY=django-insecure-#nvkx1%+=m5nb9g^6a4k@!@&f@d@&v3!e7^#-1h8lo#)f9r9qy
```

* em `settings.py` utilize a variável de ambiente para SECRET_KEY: 
```python
# config/settings.py

SECRET_KEY = env.str("SECRET_KEY")
```


## 6. Atualização de DATABASES para usar SQLite localmente e PostgreSQL em produção

* em ambiente de desenvolvimento local, usamos a base de dados SQLite. Mas em produção (no Heroku) devemos usar PostgreSQL, pois o ficheiro db.sqlite é apagado pelo Heroku.
* O módulo instalado environs[django] trata de todas as configurações necessárias
* atualize `settings.py` com:
```python
# config/settings.py

DATABASES = {
      "default": env.dj_db_url("DATABASE_URL") 
 }
```
* em `.env` especifique:
```
export DATABASE_URL=sqlite:///db.sqlite3
```
* Heroku cria uma base de dados nova PostgreSQL, e cria uma variavel de configuração chamada `DATABASE_URL`. Como .env não é carregado no Heroku, o nosso projeto Django usará no Heroku esta configuração PosrtgreSQL.
* com o ambiente virtual ativo, devemos instalar o adaptador de base de dados Psycopg, que põe o Python  a comunicar com bases de dados PostgreSQL.
```
> pipenv install psycopg2-binary==2.8.6
```


## 7. Configuração dos ficheiros estáticos
* Django não suporta o "serviço" de ficheiros staticos em produção
* Devemos instalar o pacote `whitenoise` para *static file hosting*:
```
> pipenv install whitenoise==5.1.0
```
* Em `settings.py` adicione as seguintes instruções (nas posições relativas identificadas com o comentario `# novo`). 
* Pormenor importante: em STATICFILES_DIRS, deve especificar onde se encontram as pastas static da(s) aplicação(ões) do projeto. Ou seja, tipicamente terá `STATICFILES_DIRS = [str(BASE_DIR.joinpath('app/static'))]` onde app é o nome de cada aplicação:
```python
# config/settings.py

INSTALLED_APPS = [
    ...
    'whitenoise.runserver_nostatic',  # novo   
    'django.contrib.staticfiles',
]


MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',        
    'django.contrib.sessions.middleware.SessionMiddleware',
    'whitenoise.middleware.WhiteNoiseMiddleware',    # novo
    ...
]

STATIC_URL = '/static/'
STATICFILES_DIRS = [str(BASE_DIR.joinpath('static'))]  # novo se a pasta static estiver na pasta da aplicação app, altere para str(BASE_DIR.joinpath('app/static'))
STATIC_ROOT = str(BASE_DIR.joinpath('staticfiles'))   # novo 
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'  # novo

```

* corra, com o ambiente virtual ativo, o comando `collectstatic` para compilar todas as pastas e ficheiros estaticos numa unidade para deployment:
```
> python manage.py collectstatic
```
* em `base.html` inclua no início a etiqueta {% load static %}, para que os templates incluam ficheiros estaticos:
```html
<!-- templates/base.html -->
{% load static %}
<html>
...
```

## 8. Instalação de `gunicorn`, servidor web de produção
* com o ambiente virtual ativo, instale Gunicorn como o servidow web de produção:
```
> pipenv install gunicorn==19.9.0
```
* Crie o ficheiro `Procfile`, na pasta onde está o `manage.py`, que informa o Heroku que usará gunicorn (especifique, antes de `.wsgi`, o nome do projeto; neste caso `config`):
```
web: gunicorn config.wsgi --log-file -
```
## 9. Git & GitHub

* Inicialize o repositorio Git, com ambiente virtual ativo:
```
> git status
> git add -A
> git commit -m "commit inicial"
```
* crie um repositório no GitHub para guardar o código e utilize o URL deste para fazer comandos finais (substitua pelo inserido `https://github.com/username/repositorio`):
```
> git remote add origin https://github.com/username/repositorio
> git push -u origin master
```

## 10. Deployment no Heroku
* Crie a aplicação Heroku:
```
> heroku login
> heroku create minha-app-heroku
> heroku git:remote -a minha-app-heroku
> heroku addons:create heroku-postgresql:hobby-dev
```
* o último comando verá que indica que cria um valor para a variável DATABASE_URL
* devemos definir apenas a SECRET_KEY, copiando de .env e correndo o comando seguinte colocando a SECRET_KEY entre plicas ''; se na SECRET_KEY tiver carateres `&` e `)`, coloque-os entre "":
```
> heroku config:set SECRET_KEY='django-insecure-#nvkx1%+=m5nb9g^6a4k@!@&f@d@&v3!e7^#-1h8lo#)f9r9qy'
> git push heroku master
> heroku ps:scale web=1
```
* aparecerá o URL da sua aplicação, ou pode lançá-la com o comando `heroku open`. Não corre (aparecerá `Server Error 500`) pois ainda precisa migrar para a nova base de dados no Heroku:
```
> heroku run python manage.py migrate
> heroku run python manage.py createsuperuser
```
* deverá criar novos dados atraves do modo admin, pois é uma base de dados nova.
* Para sites maiores, [fixtures](https://docs.djangoproject.com/en/3.1/howto/initial-data/) permitem carregar dados  

## Referências
* William Vincent, *Django for beginners*, 2020
* DjangoProject, *[Deployment checklist](https://docs.djangoproject.com/en/3.2/howto/deployment/checklist/)*
* [learndjango](https://learndjango.com/)
* Heroku, *Django assets* (https://devcenter.heroku.com/articles/django-assets)
<!-- 
## Outras referências
* https://django-environ.readthedocs.io/en/latest/
* https://stackoverflow.com/questions/50322966/changing-django-development-database-from-the-default-sqlite-to-postgresql
* django-heroku
* https://alicecampkin.medium.com/how-to-set-up-environment-variables-in-django-f3c4db78c55f
-->
