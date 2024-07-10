---
title: Data System API Integration Script
summary: An advanced script designed to automate the retrieval and processing of sales and exchanges data from the API. It utilizes pagination, data transformation, and Excel integration to provide comprehensive data management for sales transactions and product exchanges.
tags:
  - API
date: 2024-07-10
external_link: 
---
### Project Description: Data System API Integration Script

**Overview:**
This project involves a Python script that automates the retrieval and processing of sales and exchanges data from an API. It covers various aspects of data collection, including authentication, data transformation, and storage in Excel files.

**Key Features:**
1. **API Integration:**
   - Connects to a specific API to fetch sales and exchange data using pagination to handle large datasets.

2. **Data Transformation:**
   - Processes the retrieved data to extract relevant details and transforms it into a structured DataFrame.

3. **Payment Methods Processing:**
   - Extracts and processes information related to payment methods for each transaction.

4. **Excel Integration:**
   - Stores the processed data in Excel files for easy access and analysis, ensuring data integrity by removing duplicates and handling missing data.

**Purpose:**
This script is an essential tool for businesses that need to manage and analyze sales and exchange data efficiently. It reduces manual effort, enhances data accuracy, and ensures comprehensive data management.

**Source Code:**

```python
import requests
import json
import pandas as pd
import sys
import warnings
from datetime import datetime, timedelta

warnings.filterwarnings("ignore")
requests.packages.urllib3.disable_warnings()

class Authenticator:
    def __init__(self, url):
        self.url = url
        self.token = None

    def get_token(self):
        """Recupera o token de autenticação da API."""
        body = {
            "cnpj": "SEU_CNPJ_AQUI",
            "hash": "SEU_HASH_AQUI"
        }
        response = requests.post(f'{self.url}/v1/autenticar', json=body, verify=False)
        if response.status_code != 200:
            raise Exception(f'Erro ao tentar pegar token: {response.text}')
        self.token = response.json().get('token')
        return self.token

class APIClient:
    def __init__(self, base_url, token):
        self.base_url = base_url
        self.token = token

    def fetch_data(self, page, endpoint, params):
        """Busca dados de um endpoint específico usando paginação."""
        headers = {'Authorization': f'Bearer {self.token}'}
        params["pagina"] = page
        response = requests.get(f'{self.base_url}{endpoint}', headers=headers, params=params, verify=False)
        return response.json() if response.status_code == 200 else {}

class DataProcessor:
    @staticmethod
    def process_forma_pagamento(parcelas, codigo):
        """Processa 'parcelas' para extrair informações de forma de pagamento."""
        df_parcelas = pd.DataFrame(parcelas)
        if df_parcelas.empty:
            return pd.DataFrame()
    
        df_parcelas['formaPagamento'] = df_parcelas['forma'].map({
            1: "A VISTA", 2: "CREDIÁRIO", 4: "CARTÃO"
        })
    
        # Ajuste na expressão regular para extrair diferentes formas de pagamento corretamente
        df_parcelas['vezes'] = df_parcelas['descricao'].str.extract(r'(\d+X|DÉBITO|PIX|DINHEIRO|DEBITO)', expand=False)
    
        df_parcelas['formaPgto'] = df_parcelas['formaPagamento'].fillna('') + " " + df_parcelas['vezes'].fillna('')
        df_parcelas['formaPgto'] = df_parcelas['formaPgto'].str.strip().replace({
            "CARTÃO PIX": "PIX",
            "CARTÃO DEBITO": "CARTÃO DÉBITO"
        })
    
        df_parcelas = df_parcelas[['formaPgto']].drop_duplicates()
        df_parcelas['codigoVenda'] = codigo
        return df_parcelas

    @staticmethod
    def process_item(transaction, item, nota_fiscal_items):
        """Processa um item de uma transação para extrair informações relevantes."""
        item_id_original = item['item']
        item_index = transaction['itens'].index(item) + 1

        emissao_gerencial = datetime.strptime(transaction['notaFiscal']['dataemissao'], "%Y-%m-%dT%H:%M:%S").strftime("%Y-%m-%d %H:%M:%S")
        data_emissao = datetime.strptime(transaction['notaFiscal']['dataemissao'], "%Y-%m-%dT%H:%M:%S").strftime("%Y-%m-%d")
        hora = datetime.strptime(transaction['hora'], "%H.%M").strftime("%H:%M:%S")
        hora_envio_nf = datetime.strptime(transaction['notaFiscal']['horaenvionfe'], "%H:%M:%S").strftime("%H:%M:%S")
        data_hora_envio_nf = f"{data_emissao} {hora_envio_nf}"

        nota_fiscal_item = next((nfi for nfi in nota_fiscal_items if str(nfi['item']) == str(item_index)), {})
        
        row = {
            'nota': transaction['nota'],
            'serieNf': transaction['serieNf'],
            'chavenfe': transaction['notaFiscal']['chavenfe'],
            'statusnfe': transaction['notaFiscal']['statusnfe'],
            'cfop': transaction['notaFiscal']['cfop'],
            'loja': transaction['loja'],
            'emissaoGerencial': emissao_gerencial,
            'dataemissao': data_emissao,
            'hora': hora,
            'erpcodigopai': item['erpcodigopai'],
            'erpcodigofilho': item['erpcodigofilho'],
            'cpfCliente': transaction['cliente']['cpf'],
            'cpfVendedor': item.get('vendedor', {}).get('cpf', ''),
            'nomeVendedor': item.get('vendedor', {}).get('nome', ''),
            'item_id': item_index,  # Usar o novo index reajustado
            'quantidade': item['qtde'],
            'valorbruto': item['valor'],
            'desconto': nota_fiscal_item.get('desconto', 0),
            'vlroutras': nota_fiscal_item.get('vlroutras', 0),
            'valorLiquido': item['valor'] + nota_fiscal_item.get('vlroutras', 0) - nota_fiscal_item.get('desconto', 0),
            'dataprocessamento': data_emissao,
            'horaenvionfe': hora_envio_nf,
            'chavereferenciada': '',
            'formaPgto': DataProcessor.process_forma_pagamento(transaction.get('parcelas', []), transaction['nota'])['formaPgto'].iloc[0] if not DataProcessor.process_forma_pagamento(transaction.get('parcelas', []), transaction['nota']).empty else '',
            'tamanho': item['tamanho'],
            'vd': 'V',
            'origem': 'USE',
            'valorProduto': item['valor'] + nota_fiscal_item.get('vlroutras', 0),
            'produto': '',
            'status': '',
            'situacao': '',
            'datahoraemissao': data_hora_envio_nf
        }
        return row

    @staticmethod
    def extract_relevant_data(json_data):
        """Extrai dados relevantes das transações."""
        rows = []
        for transaction in json_data.get('itens', []):
            nota_fiscal_items = transaction.get('itensNotaFiscal', [])
            for item in transaction.get('itens', []):
                row = DataProcessor.process_item(transaction, item, nota_fiscal_items)
                rows.append(row)
        return pd.DataFrame(rows)
    
    @staticmethod
    def adicionar_chave_nova(df, colunas, separador="_"):
        """Adiciona uma nova chave combinada a partir de várias colunas."""
        df['chave_nova'] = ''
        for coluna, zfill in colunas:
            if zfill:
                df['chave_nova'] += df[coluna].astype(
                    str).str.zfill(zfill).str.strip() + separador
            else:
                df['chave_nova'] += df[coluna].astype(
                    str).str.strip() + separador
        df['chave_nova'] = df['chave_nova'].str.rstrip(separador)
        return df
    
class VendaDataHandler:
    def __init__(self, api_client, filial_inicial, filial_final):
        self.api_client = api_client
        self.filial_inicial = filial_inicial
        self.filial_final = filial_final

    def get_vendas(self, data_inicial, data_final):
        """Busca vendas de todas as filiais no intervalo de datas especificado."""
        print(f"Buscando vendas de {data_inicial.strftime('%d/%m/%Y')} até {data_final.strftime('%d/%m/%Y')}")
        all_data = []
        for filial in range(self.filial_inicial, self.filial_final + 1):
            params = {
                "loja": filial,
                "dataVendaInicio": data_inicial.strftime("%Y-%m-%d"),
                "horaVendaInicio": "00:00",
                "dataVendaFim": data_final.strftime("%Y-%m-%d"),
                "horaVendaFim": "23:59",
                "campoOrdem": "DATULTALT",
                "ordem": "ASC",
                "itensPorPagina": 1000,
                "pagina": 1               
            }
            page = 1
            while True:
                data = self.api_client.fetch_data(page, "/v1/personalizado-1/monjua/vendas", params)
                if not data.get('itens'):
                    break
                all_data.extend(data['itens'])
                page += 1

        df = DataProcessor.extract_relevant_data({'itens': all_data})
        print(f"Total de vendas: {len(df)}")
        return df
    
    def get_trocas(self, data_inicial, data_final):
        """Busca trocas de todas as filiais no intervalo de datas especificado."""
        print(f"Buscando trocas de {data_inicial.strftime('%d/%m/%Y')} até {data_final.strftime('%d/%m/%Y')}")
        all_data = []
        for filial in range(self.filial_inicial, self.filial_final + 1):
            params = {
                "loja": filial,
                "datatrocaInicio": data_inicial.strftime("%Y-%m-%d"),
                "datatrocaFim": data_final.strftime("%Y-%m-%d"),
                "itensPorPagina": 1000,
                "pagina": 1
            }
            page = 1
            while True:
                data = self.api_client.fetch_data(page, "/v1/personalizado-1/monjua/trocas", params)
                if not data.get('itens'):
                    break
                all_data.extend(data['itens'])
                page += 1
    
        rows = []
        for transaction in all_data:
            if 'movimentofiscal' in transaction:
                for mov_fiscal in transaction['movimentofiscal']:
                    for troca_item in transaction.get('itenstroca', []):
                        item_fiscal = next((item for item in transaction.get('itensmovimentofiscal', []) if str(item['item']) == str(troca_item['item'])), None)
                        if item_fiscal:
                            emissao_gerencial = datetime.strptime(mov_fiscal['dataemissao'], "%Y-%m-%dT%H:%M:%S").strftime("%Y-%m-%d %H:%M:%S")
                            data_emissao = datetime.strptime(mov_fiscal['dataemissao'], "%Y-%m-%dT%H:%M:%S").strftime("%Y-%m-%d")
                            hora_envio_nf = datetime.strptime(mov_fiscal['horaenvionfe'], "%H:%M:%S").strftime("%H:%M:%S")
                            data_hora_envio_nf = f"{data_emissao} {hora_envio_nf}"
                            row = {
                                'nota': mov_fiscal['nota'],
                                'serieNf': mov_fiscal['serie'],
                                'chavenfe': mov_fiscal['chavenfe'],
                                'statusnfe': mov_fiscal['statusnfe'],
                                'cfop': mov_fiscal['cfop'],
                                'loja': transaction['loja'],
                                'emissaoGerencial': emissao_gerencial,
                                'dataemissao': data_emissao,
                                'hora': mov_fiscal['horaenvionfe'],
                                'erpcodigopai': troca_item['erpcodigopai'],
                                'erpcodigofilho': troca_item['erpcodigofilho'],
                                'cpfCliente': transaction['cliente_cpf'],
                                'cpfVendedor': transaction['funcionario_cpf'],
                                'nomeVendedor': '',
                                'item_id': troca_item['item'],
                                'quantidade': troca_item['qtde'],
                                'valorbruto': troca_item['total'],
                                'desconto': troca_item['desconto'],
                                'vlroutras': troca_item.get('vlroutras', 0),
                                'valorLiquido': troca_item['total'] - troca_item['desconto'] + troca_item.get('vlroutras', 0),
                                'dataprocessamento': data_emissao,
                                'horaenvionfe': hora_envio_nf,
                                'chavereferenciada': troca_item.get('chaveNotaOrigem', ''),
                                'formaPgto': '',
                                'tamanho': troca_item.get('tamanho', ''),
                                'vd': 'D',
                                'origem': 'USE',
                                'valorProduto': troca_item['prcvenda'],
                                'produto': troca_item.get('produto', ''),
                                'status': '',
                                'situacao': '',
                                'datahoraemissao': data_hora_envio_nf
                            }
                            rows.append(row)
    
        df_trocas = pd.DataFrame(rows)
        print(f"Total de trocas: {len(df_trocas)}")
        return df_trocas
    
    def get_vendas_by_nota(self, nota):
        """Busca vendas específicas por nota fiscal."""
        params = {"nota": nota}
        data = self.api_client.fetch_data(1, "/v1/personalizado-1/monjua/vendas", params)
        return data
    
class Main:
    def __init__(self, dias_subtrair=0, filialInicial=1, filialFinal=80):
        self.data_final = datetime.now().date()
        self.data_inicial = self.data_final - timedelta(days=int(dias_subtrair))
        self.filial_inicial = filialInicial
        self.filial_final = filialFinal
        self.setup()

    def setup(self):
        """Configura a autenticação e o cliente da API."""
        self.authenticator = Authenticator('https://integracaodshomologacao.useserver.com.br/api')
        token = self.authenticator.get_token()
        self.api_client = APIClient('https://integracaodshomologacao.useserver.com.br/api', token)
        self.venda_handler = VendaDataHandler(self.api_client, self.filial_inicial, self.filial_final)
        
    def update_missing_values(self, df):
        """Atualiza valores faltantes no DataFrame."""
        zero_value_rows = df[(df['valorLiquido'] == 0) & (df['valorbruto'] == 0) & (df['vd'] == 'V')]
        for index, row in zero_value_rows.iterrows():
            nota = row['nota']
            item_id = row['item']
            response = self.venda_handler.get_vendas_by_nota(nota)
            if 'itens' in response:
                for item in response['itens']:
                    if str(item['item']) == str(item_id):
                        df.at[index, 'valorLiquido'] = item.get('valor', 0)
                        df.at[index, 'valorbruto'] = item.get('valorbruto', 0)
                        print(f"Atualizado item {item_id} da nota {nota} com novos valores.")
            else:
                print(f"Não foram encontrados itens para a nota {nota}.")
        return df
    
    def run(self):
        """Executa a coleta e processamento de dados de vendas e trocas."""
        df_vendas = self.venda_handler.get_vendas(self.data_inicial, self.data_final)
        df_trocas = self.venda_handler.get_trocas(self.data_inicial, self.data_final)

        columns_to_negate = ['quantidade', 'valorLiquido', 'valorProduto']
        for col in columns_to_negate:
            df_trocas[col] = -df_trocas[col].abs()
            
        df_total = pd.concat([df_vendas, df_trocas], ignore_index=True)
        df_total = df_total.rename(columns={'item_id': 'item'})
        df_total['loja'] = pd.to_numeric(df_total['loja'], errors='coerce').fillna(0).astype(int)
        df_total['nota'] = pd.to_numeric(df_total['nota'], errors='coerce').fillna(0).astype(int)
        
        df_total['erpcodigopai'] = pd.to_numeric(df_total['erpcodigopai'], errors='coerce').fillna(0).astype(int)
        df_total['erpcodigofilho'] = pd.to_numeric(df_total['erpcodigofilho'], errors='coerce').fillna(0).astype(int)
        
        df_total_sorted = df_total.sort_values(by=['vd', 'dataemissao', 'loja'], ascending=[False, True, True])
        
        colunas = [
            ('vd', None),
            ('dataemissao', None),
            ('loja', 2),
            ('serieNf', 4),
            ('nota', 4),
            ('item', 2)
        ]
        data_processor = DataProcessor()
        df_final = data_processor.adicionar_chave_nova(df_total_sorted, colunas)
        df_final.drop_duplicates()
        df_final = df_final[df_final['chavenfe'].notna()]
        df_final = df_final[(df_final['statusnfe'] == 1) | (df_final['statusnfe'] == 3)]
        df_final["status"] = 1
        df_final.loc[df_final['statusnfe'] == 1, "situacao"] = 2
        df_final.loc[df_final['statusnfe'] == 3, "situacao"] = 1
        
        condicao = (df_final['valorbruto'] != df_final['valorProduto']) & (df_final['desconto'] == 0) & (df_final['vlroutras'] == 0)
        df_final.loc[condicao, 'valorbruto'] = df_final['valorProduto']
        df_final.loc[condicao, 'valorLiquido'] = df_final['valorProduto']
        
        df_final = df_final[df_final['statusnfe'] != 1]

        nomeArquivo = "stgVendasUSE.xlsx"  
        with pd.ExcelWriter(nomeArquivo) as writer:
            df_final.to_excel(writer, index=False, sheet_name='vendas')
        return df_final

if __name__ == "__main__":
    if len(sys.argv) > 1:
        dias_subtrair = sys.argv[1]
    else:
        dias_subtrair = 1
    main = Main(dias_subtrair)
    main.run()