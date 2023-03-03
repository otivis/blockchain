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

