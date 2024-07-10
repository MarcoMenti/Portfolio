---
title: Mercado Livre API Integration Script
summary: A sophisticated script integrating with the Mercado Livre API to automate the retrieval and processing of sales and returns data. The script leverages Google Sheets for token management and uses Python for data processing and storage in Excel files.
tags:
  - API
date: 2024-07-10
external_link: 
---
### Project Description: Mercado Livre API Integration Script

**Overview:**
This project involves a Python script designed to integrate with the Mercado Livre API, automating the retrieval and processing of sales and returns data. The script uses Google Sheets for managing the API access token, which is refreshed every four hours, and stores the processed data in Excel files.

**Key Features:**
1. **API Integration:**
   - Connects to the Mercado Livre API to fetch sales and return data.

2. **Token Management:**
   - Uses Google Sheets to manage and refresh the API access token every four hours, avoiding the complexity of automating through Selenium.

3. **Sales and Returns Management:**
   - Retrieves detailed sales and return information, including item details, shipping data, and discount amounts.

4. **Data Processing:**
   - Processes the retrieved data to extract relevant details and compiles it into structured DataFrames.

5. **Excel Integration:**
   - Stores the processed data in Excel files for easy access and analysis. Ensures data integrity by removing duplicates and handling missing data.

**Purpose:**
This script is an essential tool for sellers on Mercado Livre, automating the tedious process of data retrieval and processing. It enhances efficiency, reduces manual effort, and ensures accurate data management.

**Source Code:**

```python
import gspread
import json
import requests
import pandas as pd
from oauth2client.service_account import ServiceAccountCredentials
from datetime import datetime, timedelta

def get_token():
    # Coloque aqui suas credenciais do JSON obtido do Google Sheets
    credentials_json = """
    {
      "type": "service_account",
      "project_id": "coloque aqui suas credenciais do json obtido do google sheets",
      "private_key_id": "coloque aqui suas credenciais do json obtido do google sheets",
      "private_key": "coloque aqui suas credenciais do json obtido do google sheets",
      "client_email": "coloque aqui suas credenciais do json obtido do google sheets",
      "client_id": "coloque aqui suas credenciais do json obtido do google sheets",
      "auth_uri": "coloque aqui suas credenciais do json obtido do google sheets",
      "token_uri": "coloque aqui suas credenciais do json obtido do google sheets",
      "auth_provider_x509_cert_url": "coloque aqui suas credenciais do json obtido do google sheets",
      "client_x509_cert_url": "coloque aqui suas credenciais do json obtido do google sheets"
    }
    """
    
    credentials_dict = json.loads(credentials_json)
    
    # Definir o escopo
    scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
    
    # Credenciais para acessar a planilha
    creds = ServiceAccountCredentials.from_json_keyfile_dict(credentials_dict, scope)
    client = gspread.authorize(creds)
    
    # Acessar a planilha pelo nome ou pelo ID
    spreadsheet = client.open('API-ML')
    
    # Acessar a aba específica da planilha
    sheet = spreadsheet.get_worksheet(1)
    
    # Obter o token de acesso
    access_token = sheet.cell(2, 2).value

    return access_token

class MercadoLivreAPI:
    def __init__(self, access_token):
        self.access_token = access_token
        self.user_id = self.get_user_id()

    def get_user_id(self):
        """Obtém o user_id do vendedor a partir do token de acesso."""
        url = 'https://api.mercadolibre.com/users/me'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            user_data = response.json()
            return user_data['id']
        else:
            print(f"Erro ao obter user_id: {response.status_code}")
            return None
    
    def get_sales(self, date_from=None, date_to=None, invoice_status=None, exclude_cancelled=True):
        """Obtém as vendas do Mercado Livre."""
        if not self.user_id:
            print("user_id não disponível. Verifique o access_token.")
            return None
        
        sales_data = []
        limit = 50
        offset = 0
        
        while True:
            url = f'https://api.mercadolibre.com/orders/search?seller={self.user_id}'
            headers = {'Authorization': f'Bearer {self.access_token}'}
            params = {'limit': limit, 'offset': offset}
            
            if date_from:
                params['order.date_created.from'] = date_from
            if date_to:
                params['order.date_created.to'] = date_to
            if invoice_status:
                params['invoice_status'] = invoice_status
            if exclude_cancelled:
                params['order.status'] = 'paid'
            
            response = requests.get(url, headers=headers, params=params)
            
            if response.status_code == 200:
                result = response.json()
                sales_data.extend(result['results'])
                if len(result['results']) < limit:
                    break
                offset += limit
            else:
                print(f"Erro ao obter vendas: {response.status_code}")
                break
        
        return sales_data
    
    def get_order_details(self, order_id):
        """Obtém os detalhes de um pedido específico."""
        url = f'https://api.mercadolibre.com/orders/{order_id}'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            return response.json()
        else:
            print(f"Erro ao obter detalhes do pedido: {response.status_code}")
            return None
    
    def get_item_details(self, item_id):
        """Obtém os detalhes de um item específico."""
        url = f'https://api.mercadolibre.com/items/{item_id}'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            return response.json()
        else:
            print(f"Erro ao obter detalhes do item: {response.status_code}")
            return None

    def get_shipping_details(self, shipping_id):
        """Obtém os detalhes do envio de um pedido."""
        url = f'https://api.mercadolibre.com/shipments/{shipping_id}'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            return response.json()
        else:
            print(f"Erro ao obter detalhes do envio: {response.status_code}")
            return None

    def get_invoice_details(self, user_id, order_id):
        """Obtém os detalhes da nota fiscal de um pedido."""
        url = f'https://api.mercadolibre.com/users/{user_id}/invoices/orders/{order_id}'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            return response.json()
        else:
            print(f"Erro ao obter detalhes da nota fiscal: {response.status_code}")
            return None
        
    def get_discounts(self, order_id):
        """Obtém os descontos aplicados a um pedido."""
        url = f'https://api.mercadolibre.com/orders/{order_id}/discounts'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            discount_data = response.json()
            total_discount = 0
            for detail in discount_data.get('details', []):
                for item in detail.get('items', []):
                    total_discount += item.get('amounts', {}).get('total', 0)
            return total_discount
        else:
            print(f"Erro ao obter detalhes dos descontos: {response.status_code}")
            return 0
        
    def get_claims(self):
        """Obtém as reclamações (claims) fechadas."""
        claims_data = []
        limit = 30
        offset = 0
        
        while True:
            url = f'https://api.mercadolibre.com/post-purchase/v1/claims/search?status=closed&limit={limit}&offset={offset}'
            headers = {'Authorization': f'Bearer {self.access_token}'}
            response = requests.get(url, headers=headers)
            
            if response.status_code == 200:
                result = response.json().get('data', [])
                claims_data.extend(result)
                if len(result) < limit:
                    break
                offset += limit
            else:
                print(f"Erro ao obter reclamações: {response.status_code}")
                break
        
        return claims_data   
        
    def get_return_items(self, shipment_id):
        """Obtém os itens de um envio específico."""
        url = f'https://api.mercadolibre.com/shipments/{shipment_id}/items'
        headers = {'Authorization': f'Bearer {self.access_token}'}
        response = requests.get(url, headers=headers)
        
        if response.status_code == 200:
            return response.json()
        else:
            print(f"Erro ao obter itens do envio {shipment_id}: {response.status_code}")
            return []
        
    def get_latest_date_from_excel(self, file_path, days=1):
        """Obtém a data mais recente de um arquivo Excel."""
        df = pd.read_excel(file_path, engine='openpyxl')
        df['Data emissao'] = pd.to_datetime(df['Data emissao'], dayfirst=True)
        df = df.sort_values(by='Data emissao', ascending=False)
    
        if not df.empty:
            latest_date = df['Data emissao'].iloc[0] - timedelta(days)
        else:
            latest_date = datetime.now() - timedelta(days)
    
        formatted_date = latest_date.strftime('%Y-%m-%dT%H:%M:%S.000Z')
    
        return formatted_date   
    
    def update_missing_data(self, file_path):
        """Atualiza dados faltantes em um arquivo Excel."""
        df = pd.read_excel(file_path, engine='openpyxl')
        
        missing_data_df = df[(~df['Serie'].isin(['2', 'Full']) | df['Serie'].isnull()) | df['Numero Nota'].isnull()]

        for idx, row in missing_data_df.iterrows():
            order_id = row['Order ID']
            invoice_details = self.get_invoice_details(self.user_id, order_id)
            
            if invoice_details:
                df.at[idx, 'Numero Nota'] = invoice_details.get('invoice_number', '')
                df.at[idx, 'Serie'] = invoice_details.get('invoice_series', '')
                df.at[idx, 'Natureza Operacao'] = invoice_details.get('fiscal_data', {}).get('transaction_type_description', '')
                if pd.isnull(row['MarketPlace']):
                    df.at[idx, 'MarketPlace'] = 'MLB'
        
        df.to_excel(file_path, index=False, engine='openpyxl')

    def process_excel_file(self, file_path):
        """Processa um arquivo Excel para remover duplicatas e corrigir dados."""
        df = pd.read_excel(file_path)
    
        df = df.drop_duplicates()
    
        df['Serie'] = df['Serie'].astype(str)
        df['MarketPlace'] = df['MarketPlace'].astype(str)
        
        df.loc[df['Natureza Operacao'] == 'Venda de mercadorias', 'Serie'] = 'Full'
        df.loc[df['Natureza Operacao'] == 'Venda de mercadorias', 'MarketPlace'] = 'Full'
    
        df.loc[df['Natureza Operacao'] == 'Venda de mercadoria para consumidor final', 'MarketPlace'] = 'mercadolivre'
        
        df['Serie'] = df['Serie'].replace('nan', '')
        
        df['MarketPlace'] = df.apply(lambda row: row['MarketPlace'] if row['Natureza Operacao'] in ['Venda de mercadorias', 'Venda de mercadoria para consumidor final'] else '', axis=1)

        df['Serie'] = df['Serie'].apply(lambda x: '2' if x == '2.0' else x)
        
        df.to_excel('venda_de_mercadorias_ml.xlsx', index=None)

    def store_sales_data(self, sales_data):
        """Armazena os dados de vendas em um arquivo Excel."""
        colunas = [
            'Serie', 'Numero Nota', 'Data emissao', 'Item Descricao', 'Item Codigo',
            'Item Quantidade', 'Valor Unitario', 'Valor Total', 'Natureza Operacao',
            'Cidade', 'Uf', 'MarketPlace', 'Valor Desconto', 'Data e Hora', 'Order ID'
        ]

        rows = []
        
        for sale in sales_data:
            for item in sale['order_items']:
                order_details = self.get_order_details(sale['id'])
                item_details = self.get_item_details(item['item']['id'])
                item_code = item['item'].get('seller_sku') or item['item']['id']
                shipping_details = self.get_shipping_details(sale['shipping']['id']) if 'shipping' in sale else None
                invoice_details = self.get_invoice_details(self.user_id, sale['id'])
                discount_amount = self.get_discounts(sale['id'])
                
                city = shipping_details.get('receiver_address', {}).get('city', {}).get('name', '') if shipping_details else ''
                state = shipping_details.get('receiver_address', {}).get('state', {}).get('name', '') if shipping_details else ''
                
                invoice_number = invoice_details.get('invoice_number', '') if invoice_details else ''
                invoice_series = invoice_details.get('invoice_series', '') if invoice_details else ''
                invoice_operation = invoice_details.get('fiscal_data', {}).get('transaction_type_description') if invoice_details else ''
                
                row = {
                    'Serie': invoice_series,
                    'Numero Nota': invoice_number,
                    'Data emissao': sale['date_closed'],
                    'Item Descricao': item_details['title'] if item_details else item['item']['title'],
                    'Item Codigo': item_code,
                    'Item Quantidade': item['quantity'],
                    'Valor Unitario': item['unit_price'],
                    'Valor Total': item['unit_price'] * item['quantity'],
                    'Natureza Operacao': invoice_operation,
                    'Cidade': city,
                    'Uf': state,
                    'MarketPlace': sale['context']['site'],
                    'Valor Desconto': discount_amount,
                    'Data e Hora': sale['date_created'],
                    'Order ID': sale['id']
                }
                rows.append(row)
        
        df = pd.DataFrame(rows, columns=colunas)
        df['Data emissao'] = pd.to_datetime(df['Data emissao']).dt.strftime('%d/%m/%Y')
        
        try:
            old_df = pd.read_excel('venda_de_mercadorias_ml.xlsx')
            df_vendas = pd.concat([old_df, df])
        except Exception as e:
            df_vendas = df
            print(f"Erro ao ler a planilha existente: {e}")
        
        df_vendas = df_vendas.sort_values(by='Data emissao', ascending=False)
        df_vendas = df_vendas.drop_duplicates()
        df_vendas.to_excel('venda_de_mercadorias_ml.xlsx', index=False)

    def store_returns_data(self, claims_data):
        """Armazena os dados de devoluções em um arquivo Excel."""
        colunas = [
            'Serie', 'Numero Nota', 'Data emissao', 'Item Descricao', 'Item Codigo',
            'Item Quantidade', 'Valor Unitario', 'Valor Total', 'Natureza Operacao',
            'Cidade', 'Uf', 'MarketPlace', 'Valor Desconto', 'Data e Hora'
        ]

        rows = []
        
        for claim in claims_data:
            if claim['type'] in ['returns']:
                if claim['resource'] == 'order':
                    order_id = claim['resource_id']
                    order_details = self.get_order_details(order_id)
                elif claim['resource'] == 'shipment':
                    shipment_id = claim['resource_id']
                    items = self.get_return_items(shipment_id)
                    if not items:
                        continue
                    order_id = items[0]['order_id']
                    order_details = self.get_order_details(order_id)
                else:
                    continue

                if not order_details:
                    continue

                for item in order_details['order_items']:
                    item_details = self.get_item_details(item['item']['id'])
                    item_code = item['item'].get('seller_sku') or item['item']['id']
                    shipping_details = self.get_shipping_details(order_details['shipping']['id']) if 'shipping' in order_details else None
                    discount_amount = self.get_discounts(order_id)
                    
                    city = shipping_details.get('receiver_address', {}).get('city', {}).get('name', '') if shipping_details else ''
                    state = shipping_details.get('receiver_address', {}).get('state', {}).get('name', '') if shipping_details else ''
                    
                    row = {
                        'Serie': '',
                        'Numero Nota': '',
                        'Data emissao': claim['date_created'],
                        'Item Descricao': item_details['title'] if item_details else item['item']['title'],
                        'Item Codigo': item_code,
                        'Item Quantidade': item['quantity'],
                        'Valor Unitario': item['unit_price'],
                        'Valor Total': item['unit_price'] * item['quantity'],
                        'Natureza Operacao': 'Devolução de mercadorias',
                        'Cidade': city,
                        'Uf': state,
                        'MarketPlace': claim['site_id'],
                        'Valor Desconto': discount_amount,
                        'Data e Hora': claim['date_created']
                    }
                    rows.append(row)
        
        df = pd.DataFrame(rows, columns=colunas)
        
        df['Data emissao'] = pd.to_datetime(df['Data emissao']).dt.strftime('%d/%m/%Y')
        
        try:
            old_df = pd.read_excel('devolucao_de_mercadorias_ml.xlsx')
            df_devolucoes = pd.concat([old_df, df])
        except Exception as e:
            df_devolucoes = df
            print(f"Erro ao ler a planilha existente: {e}")
        
        df_devolucoes = df_devolucoes.sort_values(by='Data emissao', ascending=False)
        df_devolucoes = df_devolucoes.drop_duplicates()
        df_devolucoes.to_excel('devolucao_de_mercadorias_ml.xlsx', index=False)
        
    def vendas(self, date_from=None, date_to=None, modo=0):
        """Obtém as vendas e retorna os dados de vendas."""
        if modo == 0:
            date_from = self.get_latest_date_from_excel('venda_de_mercadorias_ml.xlsx')
        else:
            pass
    
        if date_from and date_to:
            df = self.get_sales(date_from, date_to)
        elif date_from:
            df = self.get_sales(date_from)
        else:
            df = self.get_sales()     
     
        return df
    
    def limpar_dados(self, arquivo_excel, coluna_chave):
        """Limpa dados duplicados e vazios em um arquivo Excel."""
        df = pd.read_excel(arquivo_excel)
    
        df = df.dropna(subset=[coluna_chave])
    
        df.drop_duplicates(inplace=True)
        df.to_excel(f'{arquivo_excel}', index=None)

print("Autenticando...")
access_token = get_token()
print("Token Obtido...")

ml_api = MercadoLivreAPI(access_token)

print("Obtendo Vendas...")
sales = ml_api.vendas()
if sales:
    ml_api.store_sales_data(sales)
    ml_api.update_missing_data('venda_de_mercadorias_ml.xlsx')
    ml_api.process_excel_file('venda_de_mercadorias_ml.xlsx')
print("Processo Finalizado.")

print("Obtendo Devoluções...")
claims = ml_api.get_claims()
if claims:
    ml_api.store_returns_data(claims)
print("Processo Finalizado.")

ml_api.limpar_dados('venda_de_mercadorias_ml.xlsx', 'Numero Nota')