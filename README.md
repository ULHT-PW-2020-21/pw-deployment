# 10 passos para implantação duma aplicação Django num ambiente de produção no Heroku 🏭

* Para tornar uma aplicação visível na Internet temos que fazer implantação (*deployment*) do código num servidor e base de dados externa. Chama-se isto pôr o código em ambiente de produção.
* Várias coisas devem ser feitas. 
    * Várias configurações devem ser adaptadas caso estejamos em ambiente de produção
    * Devemos usar um serviço de Web hosting (usaremos Heroku)
    * O servidor Web do Django para uso local é básico, não serve para produção (usaremos gunicorn). 
    * A base de dados SQLite só serve em ambiente de desenvolvimento, devendo ser usada uma outra (usaremos PostgreSQL).
    * Os ficheiros estáticos devem ser configurados para ambiente produção.
* A seguir detalham-se todos os passos necessários.
* **Exemplo duma aplicação que foi criada seguindo estes passos: [GitHub](https://github.com/ULHT-PW-2020-21/pw-aula-django-02-deployed-in-heroku), [Heroku](http://pw-aula03.herokuapp.com/)**.

## 1. .venv com variáveis do ambiente de desenvolvimento

Na diretoria do manage.py, crie o ficheiro `.env` que guardará chaves e passwords assim como configurações específicas para ambiente desenvolvimento. Serão definidas como variáveis de ambiente em `.env`, que podem depois ser usadas noutros ficheiros. Para tal:
* na linha de comando instalar environs (eventualmente deverá precisar das plicas ''):
```
pipenv shell
pipenv install 'environs[django]==8.0.0'
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


## 2. .gitignore com ficheiros que o GIT deve ignorar
* crie o ficheiro `.gitignore` na diretoria do `manage.py`, indicando os ficheiros a ser ignorados pelo GIT:
```
.env
__pycache__/
db.sqlite3
.DS_Store   # só para Mac
```
* o facto de não carregarmos `.env` para o Heroku permite que certas configurações só estejam ativas no ambiente de desenvolvimento.

## 3. Debug ativo em desenvolvimento mas em produção não 
* É necessário especificar que em desenvolvimento o modo DEBUG fique ativo, mas em produção não. 
* em `.env` insira:
```
export DEBUG=True 
```

* Em `config/settings.py` altere o valor de DEBUG:
```python
# config/settings.py
[...]

DEBUG = env.bool("DEBUG", default=False)
```

## 4. Host Heroku 
* Em `settings.py`inclua o URL da aplicação Heroku como host em ALLOWED_HOSTS:
```python
# config/settings.py
[...]

ALLOWED_HOSTS = ['.herokuapp.com', 'localhost', '127.0.0.1']
```

## 5. SECRET_KEY guardada

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
* O Heroku cria uma nova base de dados PostgreSQL, associada a uma variavel de configuração chamada `DATABASE_URL`. Como .env não é carregado no Heroku, o nosso projeto Django  no Heroku usará esta configuração PosrtgreSQL.
* com o ambiente virtual ativo, devemos instalar o adaptador de base de dados Psycopg, que põe o Python a comunicar com bases de dados PostgreSQL.
```
pipenv install psycopg2-binary==2.8.6
```
(se der erro remover `==2.8.6`)

## 7. Configuração dos ficheiros estáticos
* O Django não suporta o "serviço" de ficheiros staticos em produção
* Devemos instalar o pacote `whitenoise` para *static file hosting*:
```
pipenv install whitenoise==5.1.0
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

STATIC_URL = '/nome_aplicacao/static/'    # substitua nome_aplicacao pelo nome da sua aplicação
STATICFILES_DIRS = [str(BASE_DIR.joinpath('static'))]  # novo se a pasta static estiver na pasta da aplicação app, altere para str(BASE_DIR.joinpath('app/static'))
STATIC_ROOT = str(BASE_DIR.joinpath('staticfiles'))   # novo 
STATICFILES_STORAGE = 'whitenoise.storage.CompressedManifestStaticFilesStorage'  # novo

```

* corra, com o ambiente virtual ativo, o comando `collectstatic` para compilar todas as pastas e ficheiros estaticos numa unidade para deployment:
```
python manage.py collectstatic
```
* No seu layout de referencia/base (`layout.html` ou `base.html`) inclua no início a etiqueta {% load static %}, para que os templates incluam ficheiros estaticos:
```<!-- templates/base.html -->
<!DOCTYPE html>
{% load static %}
<html>
...
```

## 8. Instalação de `gunicorn`, servidor web de produção
* com o ambiente virtual ativo, instale Gunicorn como o servidow web de produção:
```
pipenv install gunicorn==19.9.0
```
* Crie o ficheiro `Procfile`, na pasta onde está o `manage.py`, que informa o Heroku que usará gunicorn (especifique, antes de `.wsgi`, o nome do projeto; neste caso `config`):
```
web: gunicorn config.wsgi --log-file -
```
## 9. Git & GitHub

* Inicialize o repositorio Git, com ambiente virtual ativo:
```
git add -A
git commit -m "commit inicial"
```
* crie um repositório no GitHub para guardar o código e utilize o URL deste para fazer comandos finais (substitua pelo inserido `https://github.com/username/repositorio`):
```
git remote add origin https://github.com/username/repositorio
git push -u origin master
```

## 10. Deployment no Heroku
* Crie a aplicação Heroku:
```bash
heroku login
heroku create minha-app-heroku
heroku git:remote -a minha-app-heroku
heroku addons:create heroku-postgresql:hobby-dev
```
* o último comando cria uma base de dados postgresql. Verá que na linha de comandos indica que cria um valor para a variável DATABASE_URL
* devemos definir apenas a SECRET_KEY, copiando de .env e correndo o comando seguinte colocando a SECRET_KEY entre plicas ''; se na SECRET_KEY tiver carateres `&` e `)`, coloque-os entre "":
* Depois lançamos a aplicação com o comando `heroku ps:scale web=1` , escalando-a a um 'dyno', i.e., teremos um servidor correndo a nossa aplicação.

```
heroku config:set SECRET_KEY='django-insecure-#nvkx1%+=m5nb9g^6a4k@!@&f@d@&v3!e7^#-1h8lo#)f9r9qy'
git push heroku main
heroku ps:scale web=1
```
* aparecerá o URL da sua aplicação, ou pode lançá-la com o comando `heroku open`. Não corre (aparecerá `Server Error 500`) pois ainda precisa migrar para a nova base de dados no Heroku:
```
heroku run python manage.py migrate
heroku run python manage.py createsuperuser
```
* É uma base de dados nova, sem dados.


# Extras

## Migração dos dados da base de dados local para o Heroku

Se tiver dados na sua base de dados local SQLite que queira carregar no Heroku pode usar [fixtures](https://docs.djangoproject.com/en/3.1/howto/initial-data/):
* **para evitar problemas com carateres utf-8** em ficheiros json, certifique-se que no Windows Settings\Language\Administrativa Language Settings\Change system locale\Region Settins tem ativada a opção [Use Unicode UTF-8 for worldwide language support](https://stackoverflow.com/questions/64457733/django-dumpdata-fails-on-special-characters) 
* exporte a base de dados da aplicação para um ficheiro JSON (no comando em baixo, substitua `myapp` pelo nome da sua aplicação)
```
python manage.py dumpdata myapp > database.json
```
   * pode especificar o formato e indentação do ficheiro  (no comando em baixo, substitua `nome_myapp` pelo nome da sua aplicação):
   ``` 
   py manage.py dumpdata myapp –-indent 2 –-format xml > database.xml
   ```
* carregue os dados na base de dados da sua aplicação no Heroku, em PostgreSQL (no comando em baixo, substitua `myapp` pelo nome da sua aplicação):
```
py ./manage.py dumpdata myapp | heroku run --no-tty "python ./manage.py loaddata --format=json -"
```
   * pode carregar um ficheiro noutro formato (XML por exemplo), alterando no comando anterior o atributo format para o valor xml

**Atenção que Heroku não guarda imagens e ficheiros carregados através de formulários: veja mais detalhes [aqui](https://www.dothedev.com/blog/heroku-django-store-your-uploaded-media-files-for-free/). deverá usar um serviço tal como o Cloudinary, para seu armazenamento. Veja detalhes [aqui](https://github.com/ULHT-PW/pw-photos)**

## Consultando tabelas na BD PSQL no Heroku

* No site Heroku, dashboard da app, selecionar postgresql\settings\database credentials
* copiar o comando heroku CLI e usar para aceder à BD (algo tipo `heroku pg:psql postgresql-fitted-60178 --app pictures-django-app`)
* use comandos psql para ver a BD:
   * `heroku pg:psql postgresql-fitted-60178 --app pictures-django-app` cria uma ligação com a BD 
   * `\dt` mostra as tabelas disponiveis
   * `\dt+` mostra info extra sobre as tabelas
   * `TABLE table_name;` mostra conteudo da tabela table_name
   * `TRUNCATE table_name;`apaga as linhas da tabela table_name


# Clonar a app no Heroku

* pode sempre fazer um clone da sua aplicação no Heroku da seguinte forma:
```bash
$ heroku login
$ heroku git:clone -a nome-da-sua-aplicacao
```

* Faça as alterações e carregue-as novamente com:
```bash
$ git add .
$ git commit -am "make it better"
$ git push heroku master
```


## Atualização aplicação no Heroku

Se fez alterações locais e pretende atualizar o GitHub e o Heroku:
```
git add -A
git commit -m "alterações"
git push
gut push heroku main
```

Se houver mudança na base de dados
* `python manage.py makemigrations` para criar instruções das alterações na base de dados
* `heroku run python manage.py migrate` para migrar as alterações na base de dados
* `heroku pg:psql postgresql-fitted-60178 --app nome_app_heroku` cria uma ligação com a BD (No site Heroku, dashboard da app, selecionar postgresql\settings\database credentials copiar o comando heroku CLI e usar para aceder à BD) 
* `TRUNCATE <app_name>_<table_name>;`apaga as linhas da tabela table_name que foi modificada (eventualmente desnecessário, se tabela for substituida)
* `python manage.py dumpdata myapp > database.json` para exportar BD para ficheiro JSON (no comando, alterar `myapp` para nome da aplicação)  
* `py ./manage.py dumpdata myapp | heroku run --no-tty "python ./manage.py loaddata --format=json -"` para carregar dados no ficheiro JSON na BD PSQL do Heroku (no comando, alterar `myapp` para nome da aplicação)

Mais detalhes encontra [aqui](https://medium.com/@sreebash/how-to-update-previous-deployed-project-on-heroku-c778d555cd8a) ou [ali](https://riptutorial.com/heroku/example/26719/deploying-with-git). Poderá eventualmente necessitar de definir o seu projeto remoto.

## Usando fork e branch para alterar um projeto de outros

1. Fork it!
2. Create your feature branch: `git checkout -b my-new-feature`
3. Commit your changes: `git commit -am 'Add some feature'`
4. Push to the branch: `git push origin my-new-feature`
5. Submit a pull request :D

 
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
