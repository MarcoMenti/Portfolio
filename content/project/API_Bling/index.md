---
title: Bling API Integration Script
summary: A robust script integrating with the Bling API to automate the fetching and processing of invoices, including sales, returns, and cancellations. This project uses Python and Selenium to enhance efficiency in managing invoicing data.
tags:
  - API
date: 2024-07-10
external_link: 
---
### Project Description: Bling API Integration Script

**Overview:**
This project involves a Python script designed to integrate with the Bling API to automate the fetching and processing of invoices. It covers sales, returns, and cancellations, leveraging the power of Python and Selenium to streamline data management.

**Key Features:**
1. **API Integration:**
   - Uses the Bling API to fetch and process invoices, ensuring up-to-date data retrieval.

2. **Sales and Returns Management:**
   - Fetches sales and return invoices, processes XML data, and compiles it into a structured DataFrame.

3. **Cancellation Filtering:**
   - Identifies and removes canceled invoices from the dataset, maintaining data accuracy.

4. **Automated Login and Token Retrieval:**
   - Automates the login process to Bling using Selenium and OAuth2 for token management.

5. **Data Processing:**
   - Processes XML invoice data to extract relevant details, including item descriptions, quantities, prices, and more.

6. **Excel Integration:**
   - Saves processed data to Excel files, ensuring easy access and analysis.

**Purpose:**
This script is an essential tool for businesses using Bling, automating the tedious process of data retrieval and processing. It enhances efficiency, reduces manual effort, and ensures accurate invoicing data management.

**Source Code:**

```python
import requests
import json
import pandas as pd
import time
import xml.etree.ElementTree as ET
from datetime import datetime, timedelta
from urllib.parse import urlencode
from itertools import chain
from typing import List, Dict
from selenium import webdriver
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
from requests_oauthlib import OAuth2Session

# Classe para fazer requisições à API do Bling
class ApiClient:
    def __init__(self, access_token=None):
        self.base_url = 'https://bling.com.br/Api/v3/'
        self.access_token = access_token
        self.headers = {
            'Authorization': f'Bearer {access_token}' if access_token else ''
        }

    def make_request(self, endpoint, params=None):
        """Faz uma requisição GET ao endpoint especificado com os parâmetros fornecidos."""
        url = self.base_url + endpoint
        response = requests.get(url, headers=self.headers, params=params)
        return self.handle_response(response)

    def handle_response(self, response):
        """Lida com a resposta da API, decodificando JSON ou retornando um erro."""
        if response.status_code in (200, 201):
            try:
                return response.json()
            except json.JSONDecodeError:
                return {"error": f"Erro na decodificação da resposta JSON: {response.text}"}
        else:
            return {"error": f"Erro na requisição: {response.status_code} - {response.text}"}

# Classe para gerenciar faturas (invoices) no Bling
class BlingInvoices:
    def __init__(self, api_client):
        self.api_client = api_client

    def get_latest_date_from_excel(self, file_path, days=1):
        """Obtém a data mais recente do arquivo Excel, descontando um número especificado de dias."""
        df = pd.read_excel(file_path, engine='openpyxl')
        df['Data emissao'] = pd.to_datetime(df['Data emissao'], dayfirst=True)
        df = df.sort_values(by='Data emissao', ascending=False)

        if not df.empty:
            latest_date = df['Data emissao'].iloc[0] - timedelta(days=days)
        else:
            latest_date = datetime.now() - timedelta(days=days)
            return None

        return latest_date.strftime('%Y-%m-%d')

    def fetch_invoices(self, start_date=None, end_date=None, tipo=1):
        """Busca faturas no Bling entre as datas especificadas."""
        invoices = []
        pagina = 1

        while True:
            params = {'pagina': pagina, 'tipo': tipo}
            if start_date:
                params['dataEmissaoInicial'] = start_date
            if end_date:
                params['dataEmissaoFinal'] = end_date

            result = self.api_client.make_request('nfe', params=params)
            if 'data' in result and result['data']:
                invoices.extend(result['data'])
                pagina += 1
            else:
                break

        return [dicionario['id'] for dicionario in invoices]

    def fetch_invoice_details(self, number):
        """Busca detalhes de uma fatura específica pelo número."""
        endpoint = f"nfe/{number}"
        result = self.api_client.make_request(endpoint)

        if 'error' in result:
            return None

        return {
            'xml_url': result['data'].get('xml'),
            'store_id': result['data'].get('loja', {}).get('id')
        }

    def fetch_all_invoices_details(self, invoice_numbers):
        """Busca detalhes de todas as faturas fornecidas."""
        all_details = []
        if invoice_numbers and isinstance(invoice_numbers[0], list):
            invoice_numbers = list(chain(*invoice_numbers))

        for number in invoice_numbers:
            details = self.fetch_invoice_details(number)
            if details:
                all_details.append(details)
        return all_details

    def process_xml_urls(self, invoice_details: List[Dict[str, any]]) -> List[Dict[str, any]]:
        """Processa as URLs dos XMLs das faturas e extrai informações relevantes."""
        ns = {'nfe': 'http://www.portalfiscal.inf.br/nfe'}

        for detail in invoice_details:
            try:
                response = requests.get(detail['xml_url'])
                response.raise_for_status()
                xml_content = response.text

                root = ET.fromstring(xml_content)

                ide = root.find('.//nfe:ide', ns)
                detail['natOp'] = ide.find('nfe:natOp', ns).text
                detail['serie'] = ide.find('nfe:serie', ns).text
                detail['nNF'] = ide.find('nfe:nNF', ns).text
                detail['dhEmi'] = ide.find('nfe:dhEmi', ns).text

                dest = root.find('.//nfe:dest/nfe:enderDest', ns)
                detail['destinatario'] = {
                    'xMun': dest.find('nfe:xMun', ns).text,
                    'UF': dest.find('nfe:UF', ns).text
                }

                detail['items'] = []
                for det in root.findall('.//nfe:det', ns):
                    prod = det.find('nfe:prod', ns)
                    items_detail = {
                        'cProd': prod.find('nfe:cProd', ns).text,
                        'xProd': prod.find('nfe:xProd', ns).text,
                        'qCom': prod.find('nfe:qCom', ns).text,
                        'vUnCom': prod.find('nfe:vUnCom', ns).text,
                        'vDesc': prod.find('nfe:vDesc', ns).text if prod.find('nfe:vDesc', ns) is not None else '0.00'
                    }
                    detail['items'].append(items_detail)

            except Exception as e:
                print(f"An error occurred while processing {detail['xml_url']}: {e}")
                detail['error'] = str(e)

        return invoice_details

    def create_dataframe(self, processed_data):
        """Cria um DataFrame a partir dos dados processados das faturas."""
        rows = []

        marketplace_map = {
            203326326: "Amazon Entregas",
            203329411: "Magalu Entregas",
            204900037: "Shein Envios",
            203904475: "Shopee Envios",
            203985123: "NetShoes",
            203326268: "B2W Entrega"
        }

        for invoice in processed_data:
            if 'error' not in invoice:
                marketplace_name = marketplace_map.get(invoice['store_id'], "Desconhecido")

                for item in invoice['items']:
                    dhEmi_processed = invoice['dhEmi'][:19]
                    date_emissao = datetime.strptime(dhEmi_processed, "%Y-%m-%dT%H:%M:%S").strftime("%d/%m/%Y")

                    row = {
                        'Serie': invoice['serie'],
                        'Numero Nota': invoice['nNF'],
                        'Data emissao': date_emissao,
                        'Item Descricao': item['xProd'],
                        'Item Codigo': item['cProd'],
                        'Item Quantidade': float(item['qCom']),
                        'Valor Unitario': float(item['vUnCom']),
                        'Natureza Operacao': invoice['natOp'],
                        'Cidade': invoice['destinatario']['xMun'],
                        'Uf': invoice['destinatario']['UF'],
                        'MarketPlace': marketplace_name,
                        'Valor Desconto': float(item['vDesc']),
                        'Data e Hora': invoice['dhEmi']
                    }

                    row['Valor Total'] = row['Item Quantidade'] * row['Valor Unitario']
                    rows.append(row)

        df = pd.DataFrame(rows)

        colunas = [
            'Serie', 'Numero Nota', 'Data emissao', 'Item Descricao', 'Item Codigo',
            'Item Quantidade', 'Valor Unitario', 'Valor Total', 'Natureza Operacao',
            'Cidade', 'Uf', 'MarketPlace', 'Valor Desconto', 'Data e Hora'
        ]

        df = df[colunas]

        return df

    def fetch_cancelled_invoices(self, start_date=None, end_date=None):
        """Busca faturas canceladas no Bling entre as datas especificadas."""
        invoices = []
        pagina = 1
        while True:
            params = {'pagina': pagina, 'situacao': 2}
            if start_date:
                params['dataEmissaoInicial'] = start_date
            if end_date:
                params['dataEmissaoFinal'] = end_date

            result = self.api_client.make_request('nfe', params=params)
            if 'data' in result and result['data']:
                invoices.extend(result['data'])
                pagina += 1
            else:
                break

        cancelled_invoice_numbers = [int(invoice.get('numero')) for invoice in invoices]

        df_vendas = pd.read_excel('venda_de_mercadorias.xlsx')
        df_vendas['Numero Nota'] = pd.to_numeric(df_vendas['Numero Nota'], errors='coerce')

        mask = ~df_vendas['Numero Nota'].isin(cancelled_invoice_numbers)
        df_vendas_filtered = df_vendas[mask]

        return df_vendas_filtered

def vendas(access_token, start_date=None, end_date=None, modo=0):
    """Função principal para obter e processar vendas."""
    api_client = ApiClient(access_token)
    bling_invoices = BlingInvoices(api_client)

    if modo == 0:
        start_date = bling_invoices.get_latest_date_from_excel('venda_de_mercadorias.xlsx')

    if start_date and end_date:
        df = bling_invoices.fetch_invoices(start_date, end_date)
    elif start_date:
        df = bling_invoices.fetch_invoices(start_date)
    else:
        df = bling_invoices.fetch_invoices()
        
    df2 = bling_invoices.fetch_all_invoices_details(df)
    df3 = bling_invoices.process_xml_urls(df2)
    df4 = bling_invoices.create_dataframe(df3)
    
    try:
        old_df = pd.read_excel('venda_de_mercadorias.xlsx')
        df_venda = pd.concat([old_df, df4])
    except Exception as e:
        print(f"Erro ao ler a planilha existente: {e}")
        df_venda = df4
        
    df_venda = df_venda.sort_values(by='Data emissao', ascending=False).drop_duplicates()
    df_venda.to_excel('venda_de_mercadorias.xlsx', index=False)

def devolucoes(access_token, start_date=None, end_date=None, modo=0):
    """Função principal para obter e processar devoluções."""
    api_client = ApiClient(access_token)
    bling_invoices = BlingInvoices(api_client)

    if modo == 0:
        start_date = bling_invoices.get_latest_date_from_excel('devolucao_de_mercadorias.xlsx')

    if start_date and end_date:
        df = bling_invoices.fetch_invoices(start_date, end_date, tipo=0)
    elif start_date:
        df = bling_invoices.fetch_invoices(start_date, tipo=0)
    else:
        df = bling_invoices.fetch_invoices(tipo=0)
        
    df2 = bling_invoices.fetch_all_invoices_details(df)
    df3 = bling_invoices.process_xml_urls(df2)
    df4 = bling_invoices.create_dataframe(df3)
    
    try:
        old_df = pd.read_excel('devolucao_de_mercadorias.xlsx')
        df_venda = pd.concat([old_df, df4])
    except Exception as e:
        print(f"Erro ao ler a planilha existente: {e}")
        df_venda = df4
        
    df_venda = df_venda.sort_values(by='Data emissao', ascending=False).drop_duplicates()
    df_venda.to_excel('devolucao_de_mercadorias.xlsx', index=False)

def filtrar_canceladas(access_token, start_date=None, end_date=None, modo=0):
    """Função principal para filtrar notas canceladas."""
    api_client = ApiClient(access_token)
    bling_invoices = BlingInvoices(api_client)
    
    if modo == 0:
        start_date = bling_invoices.get_latest_date_from_excel('venda_de_mercadorias.xlsx', days=10)

    if start_date and end_date:
        df = bling_invoices.fetch_cancelled_invoices(start_date, end_date)
    elif start_date:
        df = bling_invoices.fetch_cancelled_invoices(start_date)
    else:
        df = bling_invoices.fetch_cancelled_invoices()
        
    df = df.sort_values(by='Data emissao', ascending=False).drop_duplicates()
    df.to_excel('venda_de_mercadorias.xlsx', index=False)

print("Autenticando...")
client_id = 'seu client id'
client_secret = 'seu client secret'
redirect_uri = 'http://localhost:5000/callback'
authorization_base_url = 'https://www.bling.com.br/Api/v3/oauth/authorize'
token_url = 'https://www.bling.com.br/Api/v3/oauth/token'

# Autenticação OAuth
oauth = OAuth2Session(client_id, redirect_uri=redirect_uri)
authorization_url, state = oauth.authorization_url(authorization_base_url)

chrome_options = Options()
chrome_options.add_argument("--headless")
chrome_options.add_argument("--no-sandbox")
chrome_options.add_argument("--disable-gpu")
driver = webdriver.Chrome(options=chrome_options)
driver.get(authorization_url)

# Espera pelos campos de login
WebDriverWait(driver, 10).until(EC.presence_of_element_located((By.CSS_SELECTOR, "#content > div > div > form > div:nth-child(2) > input")))
username = driver.find_element(By.CSS_SELECTOR, "#content > div > div > form > div:nth-child(2) > input")
password = driver.find_element(By.CSS_SELECTOR, "#content > div > div > form > div.mdc-layout-grid__cell--span-12.password > input")

# Insere credenciais
username.send_keys("seu usuario)
password.send_keys("sua senha")

login_button = driver.find_element(By.CSS_SELECTOR, "#content > div > div > form > div.mdc-layout-grid__cell--span-12.mdc-layout-grid--align-right.submit-login > button")
login_button.click()

# Espera pelo redirecionamento
WebDriverWait(driver, 10).until(EC.url_contains(redirect_uri))
redirect_response = driver.current_url
redirect_response = redirect_response.replace("http://", "https://")

driver.quit()

# Obtém o token de acesso
token = oauth.fetch_token(token_url, authorization_response=redirect_response, client_secret=client_secret)
access_token = token['access_token']
print("Token Obtido...")

# Obtém e processa vendas
print("Obtendo Vendas...")
vendas(access_token)
print("Processo Finalizado.")

# Obtém e processa devoluções
print("Obtendo Devoluções...")
devolucoes(access_token)
print("Processo Finalizado.")

# Remove notas canceladas
print("Removendo Notas Canceladas...")
filtrar_canceladas(access_token)
print("Processo Finalizado.")

