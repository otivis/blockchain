<h1> DESAFIO IACRYPTO </H1> 
Fazer o Deploy do framework web2py o deploy precisa ser acessível via browser mesmo que em localhost

Criar uma aplicação nova.... Pode ser algum template dele mesmo, o único requisito é que essa aplicação use autenticação integrada para que eu possa me logar com minha conta do google.....

Essa aplicação deve conversar via API com o site: https://www.blockchain.com/pt/api/blockchain_api (documentação), ter um campo de pesquisa por "TXID" id (form) da transação no BlockChain que vai fazer a busca em https://blockchain.info/rawtx/$tx_hash (endereço da chmada GET JSON) e trazer os dados formatados em tabela dentro da página
    
    
<h2> Integração com a API do Blockchain </h2>

```
import requests
import json

def search():
    form = SQLFORM.factory(Field('txid', 'string', label='TXID'))
    if form.process().accepted:
        txid = form.vars.txid
        tx_data = get_data(txid)
        table_data = []
        for input_data in tx_data['inputs']:
            for prev_out in input_data['prev_out']:
                table_data.append((prev_out['addr'], prev_out['value']))
        for output_data in tx_data['out']:
            table_data.append((output_data['addr'], output_data['value']))
        return dict(table_data=table_data)
    return dict(form=form)

def get_data(txid):
    url = f"https://blockchain.info/rawtx/{txid}"
    response = requests.get(url)
    return json.loads(response.content)
    

```

<p>isso foi adicionado no arquivo default.py nos controladores da aplicação </p>

também foi criada um código "blockchain.py" dentro dos módulos da aplicação para ser importado posteriormente pelo defaul.py para ser usado.

```
import urllib.request
import json

def get_data(txid):
    url = "https://blockchain.info/rawtx/{}".format(txid)
    with urllib.request.urlopen(url) as response:
        data = json.loads(response.read().decode())
    return data
```


<h2> Páginas Front-end</h2>

Para dar usabilidade foram trocados alguns arquivos das views da aplicação por tais códigos em HTML e CSS

```
<!DOCTYPE html>
<html>
<head>
    <title>{{=response.title or request.application}}</title>
    {{if "auth" in request.controller}}
        {{include "web2py_ajax.html"}}
    {{pass}}
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    {{if "css" in response.files}}
        {{for file in response.files.css}}
            <link href="{{=URL(file)}}" type="text/css" rel="stylesheet">
        {{pass}}
    {{pass}}
    {{if "js" in response.files}}
        {{for file in response.files.js}}
            <script src="{{=URL(file)}}" type="text/javascript"></script>
        {{pass}}
    {{pass}}
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>{{=response.title or request.application}}</h1>
            <div class="menu">
                {{if auth.user_id:}}
                    <a href="{{=URL('default', 'index')}}">Home</a>
                    <a href="{{=URL('default', 'search')}}">Search</a>
                    <a href="{{=URL('default', 'logout')}}">Logout</a>
                {{else:}}
                    <a href="{{=URL('default', 'index')}}">Home</a>
                    <a href="{{=URL('default', 'login')}}">Login</a>
                    <a href="{{=URL('default', 'register')}}">Register</a>
                {{pass}}
            </div>
        </div>
        <div class="main">
            {{=response.flash}}
            {{=response.body}}
        </div>
        <div class="footer">
            {{=response.menu}}
        </div>
    </div>
</body>
</html>
```

E na pagina principal ficaria dessa forma 

```
{{extend 'layout.html'}}
{{response.title = 'Home'}}
<div class="jumbotron">
    <h1>Bem-vindo a IACRYPTO App!</h1>
    <p>Esta aplicação permite que você pesquise transações na BlockChain.</p>
    {{if not auth.user_id:}}
        <p>Clique em <a href="{{=URL('default', 'login')}}">Login</a> para começar.</p>
    {{pass}}
</div>
```
<P> Para fazer login seria desta forma </P>

```
{{extend 'layout.html'}}
{{response.title = 'Login'}}
{{if form}}
    <h2>Login</h2>
    {{=form}}
{{else:}}
    <p>Você já está logado!</p>
{{pass}}
```


<h2> Autenticação Integrada </h2>

Talvez a parte que mais me deu trabalho e funcionou apenas uma vez, antes de eu fazer a aplicação conversar com a API da Blockchain. 
Foi adicionado isso ao arquivo db.py da pasta módulos da aplicação 

```
from gluon.contrib.login_methods.oauth20_account import OAuthAccount

auth.settings.login_form = OAuthAccount(
    gauth,
    client_id='****',
    client_secret='****',
    scope=['openid', 'email', 'profile'],
    auth_url='https://accounts.google.com/o/oauth2/auth',
    token_url='https://accounts.google.com/o/oauth2/token',
    userinfo_url='https://www.googleapis.com/oauth2/v1/userinfo'
)
```

usando lib Oauth2 do google. 

<P> para solucionar tentei criar um arquivo chamado auth.py dentro da pastas controladores da aplicação com o seguinte codigo:</p>
```
from gluon.contrib.login_methods.oauth20_account import OAuthAccount
from gluon.tools import fetch
import json

def json_loads(payload):
    try:
        return json.loads(payload)
    except ValueError:
        return dict()

def auth_google():
    auth.settings.actions_disabled.append('register')
    auth.settings.login_form = OAuthAccount(
        grequests=dict(
            client_id='<SEU_CLIENT_ID>',
            client_secret='<SEU_CLIENT_SECRET>',
            scope=['openid', 'email', 'profile'],
            auth_url='https://accounts.google.com/o/oauth2/auth',
            token_url='https://accounts.google.com/o/oauth2/token',
            user_info_url='https://www.googleapis.com/oauth2/v1/userinfo',
            redirect_uri=auth.settings.login_controller,
            state='auth_provider=google',
            response_type='code',
            access_type='offline',
            approval_prompt='force'
        ),
        approval_prompt='force',
        login_url=URL('default', 'user', args='login', vars=dict(_next=URL('default', 'index')))
    )
    return dict(form=auth.login_bare())

def user():
    if request.args(0) == 'login' and request.vars._next:
        session.auth_next = request.vars._next
    elif session.auth_next:
        next = session.auth_next
        del session.auth_next
        redirect(next)
    redirect(URL('default', 'index'))

def call():
    session.forget()
    return service()

def download():
    return response.download(request, db)

def call():
    session.forget()
    return service()

def api():
    if not request.vars.txid:
        raise HTTP(400, 'missing txid parameter')
    with urllib.request.urlopen(f'https://blockchain.info/rawtx/{request.vars.txid}') as url:
        tx_data = json_loads(url.read().decode())
    if not tx_data:
        raise HTTP(404, 'transaction not found')
    tx_inputs = []
    tx_outputs = []
    for input_data in tx_data['inputs']:
        for prev_out in input_data['prev_out']:
            tx_inputs.append({'addr': prev_out['addr'], 'value': prev_out['value']})
    for output_data in tx_data['out']:
        tx_outputs.append({'addr': output_data['addr'], 'value': output_data['value']})
    response.headers['Content-Type'] = 'application/json'
    return json.dumps({'txid': request.vars.txid, 'inputs': tx_inputs, 'outputs': tx_outputs})
```
porém sem sucesso, deu uma sequência de erros relacionado ao banco de dados, o repositório esta aberto para commits no diretório Blockchain.


<H1>APLICAÇÃO FEITA UTILIZANDO WEB2PY </H1> 


