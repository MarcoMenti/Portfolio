---
title: Itau Bank Slip Automation Script
summary: An advanced automation script designed to generate bank slips for Itau Bank. The script uses Selenium to automate the process, enhancing efficiency and reducing manual effort.
tags:
  - Python
date: 2024-07-04
external_link: 
---
### Script Description: Itau Bank Slip Automation

**Overview:**
The Itau Bank Slip Automation script is designed to streamline the generation of bank slips for Itau Bank. This advanced script uses Selenium to automate the entire process, significantly enhancing efficiency and reducing manual effort.

**Key Features:**
1. **Automation with Selenium:**
   - Utilizes Selenium for web automation, ensuring a seamless and error-free generation of bank slips.

2. **Error Logging:**
   - Includes comprehensive error logging to track and resolve any issues that may arise during the automation process.

3. **Class-Based Structure:**
   - Organized using a class-based structure to improve code readability and maintainability.

4. **User Authentication:**
   - Automates the login process, securely handling user credentials to access the Itau Bank portal.

5. **Bank Slip Generation:**
   - Automatically navigates through the Itau Bank interface to generate and download bank slips, saving time and effort.

**Additional Details:**
- **Efficiency Gains:** By automating the repetitive task of generating bank slips, this script frees up valuable time for other important tasks.
- **Error Handling:** Robust error handling mechanisms are in place to ensure the script can recover from common issues without manual intervention.

**Purpose:**
This script is an essential tool for businesses and individuals who need to generate Itau Bank slips regularly. It automates a tedious process, increases productivity, and minimizes the risk of errors, making it a valuable asset for financial operations.

**Source Code:**

```python
import logging
import os
import re
import time
import random
import csv
import pandas as pd
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.chrome.options import Options
from selenium.common.exceptions import NoSuchElementException, ElementNotInteractableException
import fitz  # PyMuPDF
from PyPDF2 import PdfMerger

# Configuração do logging
logging.basicConfig(level=logging.INFO, filename='boleto_script.log', filemode='a',
                    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s')

class BoletoAutomatizador:
    def __init__(self):
        self.driver = self.configurar_navegador()

    def configurar_navegador(self):
        chrome_options = Options()
        chrome_options.add_experimental_option("prefs", {
            "download.default_directory": r"C:\Users\User\Downloads",
            "download.prompt_for_download": False,
            "download.directory_upgrade": True,
            "safebrowsing.enabled": True
        })
        chrome_options.add_argument('--remote-debugging-port=9222')
        return webdriver.Chrome(options=chrome_options)

    def acessar_pagina_e_fazer_login(self):
        try:
            self.driver.get("https://www.itau.com.br/")
            self.driver.maximize_window()
            agencia_input = WebDriverWait(self.driver, 20).until(EC.presence_of_element_located((By.XPATH, '/html/body/div[2]/header/div[2]/div/div/div/div[2]/form/div/div[1]/div[1]/input')))
            conta_input = WebDriverWait(self.driver, 20).until(EC.presence_of_element_located((By.XPATH, '/html/body/div[2]/header/div[2]/div/div/div/div[2]/form/div/div[1]/div[2]/input')))
            agencia_input.send_keys("0000")
            time.sleep(1)
            conta_input.send_keys("000000")
            time.sleep(1)
            enviar_button = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.XPATH, "/html/body/div[2]/header/div[2]/div/div/div/div[2]/form/div/div[2]/button[1]")))
            enviar_button.click()
        except Exception as e:
            logging.error("Erro ao acessar página e fazer login", exc_info=True)

    def interagir_com_teclado_virtual(self):
        try:
            teclado_virtual = WebDriverWait(self.driver, 20).until(EC.presence_of_element_located((By.CSS_SELECTOR, "#frmTecladoPessoJuridica > fieldset > div.teclado.clearfix")))
            acessar_button = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.ID, "acessar")))

            for tentativa in range(3):
                try:
                    teclado_virtual.send_keys(Keys.RETURN)
                    break
                except ElementNotInteractableException:
                    logging.warning(f"Elemento não interagível. Tentativa {tentativa + 1} falhou.")
                    time.sleep(3)

            numeros_teclado = ["0", "1", "2", "3", "4", "5"]
            actions = ActionChains(self.driver)
            for numero in numeros_teclado:
                numero_button = teclado_virtual.find_element(By.XPATH, f".//a[contains(text(), '{numero}')]")
                actions.move_to_element(numero_button).click().perform()
                time.sleep(2 + random.random())

            actions.move_to_element(acessar_button).click().perform()
            time.sleep(2 + random.random())

            # Clica no input
            time.sleep(5)
            input_acesso = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.XPATH, '/html/body/section[2]/div/section/div/div/form/div/section/fieldset/ul/li[2]/p[1]/input')))
            self.driver.execute_script("arguments[0].click();", input_acesso)
            time.sleep(1 + random.random())
            botao_acesso = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.CSS_SELECTOR, "#btn-continuar")))
            actions.move_to_element(botao_acesso).click().perform()
            time.sleep(1 + random.random())
        except Exception as e:
            logging.error("Erro ao interagir com teclado virtual", exc_info=True)

    def fechar_popups(self):
        try:
            time.sleep(15)
            close_popups(self.driver)
            close_special_popups(self.driver)
        except Exception as e:
            logging.error("Erro ao fechar popups", exc_info=True)

    def navegar_no_menu(self):
        try:
            menu_locators = [
                lambda: self.driver.find_element(By.XPATH, '//a[@aria-label="cobrança" and @class="menu-item-top"]'),
                lambda: self.driver.find_element(By.CSS_SELECTOR, '#main-menu > li:nth-child(3) > a'),
                lambda: self.driver.find_element(By.XPATH, '//*[@id="main-menu"]/li[3]/a'),
                lambda: self.driver.find_element(By.XPATH, '/html/body/div[1]/header/div[6]/div/div/nav/ul/li[3]/a')
            ]

            for locator in menu_locators:
                try:
                    menu_item = locator()
                    logging.info("Menu encontrado!")
                    break
                except NoSuchElementException:
                    logging.warning("Menu não encontrado. Tentando o próximo localizador...")

            actions = ActionChains(self.driver)
            actions.move_to_element(menu_item).perform()
            wait = WebDriverWait(self.driver, 10)
            wait.until(EC.visibility_of_element_located((By.CSS_SELECTOR, '#main-menu > li:nth-child(3) > ul')))

            # Defina seus localizadores para encontrar o item dentro do menu expandido
            item_locators = [
                lambda: self.driver.find_element(By.XPATH, '//*[@id="main-menu"]/li[3]/ul/li[1]/a'),
                lambda: self.driver.find_element(By.XPATH, '/html/body/div[1]/header/div[6]/div/div/nav/ul/li[3]/ul/li[1]/a'),
                lambda: self.driver.find_element(By.XPATH, '//a[@role="link" and @class="cursor-pointer" and @contexto="/pessoa-juridica-cobranca-app"]'),
                lambda: self.driver.find_element(By.CSS_SELECTOR, '#main-menu > li:nth-child(3) > ul > li:nth-child(1) > a')
            ]

            # Tente usar cada localizador para encontrar e clicar no item
            for locator in item_locators:
                try:
                    submenu_item = locator()
                    logging.info("Elemento encontrado!")
                    
                    # Crie uma nova instância de ActionChains e clique no item do submenu
                    actions.move_to_element(submenu_item).click().perform()
                    break
                except NoSuchElementException:
                    logging.warning("Elemento não encontrado. Tentando o próximo localizador...")
            time.sleep(5)
            
            # Fechar o pop-up se ele existir
            self.fechar_popup_emitir_boleto()
        
        except Exception as e:
            logging.error("Erro ao navegar no menu", exc_info=True)          

    def preencher_formulario_boleto(self, cnpj_cpf, valor, vencimento, fatura):
        try:
            com_pix_button = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.ID, "com_pix")))
            com_pix_button.click()
            cnpj_input = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.CSS_SELECTOR, ".clearfix #inputBuscaPagador")))
            cnpj_input.send_keys(cnpj_cpf)
            time.sleep(1)
            cnpj_input.send_keys(Keys.RETURN)
            time.sleep(5)

            ActionChains(self.driver).move_to_element(cnpj_input).click().send_keys(Keys.TAB).send_keys(Keys.TAB).send_keys(Keys.ENTER).perform()
            time.sleep(0.5)

            for _ in range(5):
                ActionChains(self.driver).send_keys(Keys.TAB).perform()
                time.sleep(0.5)

            ActionChains(self.driver).send_keys(Keys.ARROW_DOWN).send_keys(Keys.ARROW_DOWN).perform()
            time.sleep(1)

            ActionChains(self.driver).send_keys(Keys.TAB).send_keys(valor).perform()
            time.sleep(1)
            ActionChains(self.driver).send_keys(Keys.TAB).send_keys(vencimento).perform()
            time.sleep(1)
            ActionChains(self.driver).send_keys(Keys.TAB).send_keys(Keys.TAB).send_keys(Keys.TAB).send_keys(fatura).perform()
            time.sleep(1)
            ActionChains(self.driver).send_keys(Keys.TAB).send_keys(Keys.TAB).send_keys("DM").perform()
            time.sleep(1.5)
        except Exception as e:
            logging.error("Erro ao preencher formulário do boleto", exc_info=True)

    def selecionar_beneficiario(self):
        try:
            beneficiario_button = WebDriverWait(self.driver, 10).until(EC.element_to_be_clickable((By.CSS_SELECTOR, "#btnSelecionarBeneficiarioFinal")))
            actions = ActionChains(self.driver)
            actions.move_to_element(beneficiario_button).click().perform()
            time.sleep(3.5)

            beneficiario_locators = [
                lambda: self.driver.find_element(By.XPATH, '//a[contains(@class, "linkAzul") and contains(@onclick, "BENEFICIARIO FINAL")]'),
                lambda: self.driver.find_element(By.XPATH, '//a[@onclick="validaBeneficiarioIgual(\'BENEFICIARIO FINAL\', \'CNPJ DO BENEFICIARIO\', \'26f76f39-4b35-4eb1-96d1-0b1a3f4bd5fe\');"]'),
                lambda: self.driver.find_element(By.CSS_SELECTOR, '#ItauTables_Table_1 > tbody > tr.odd > td:nth-child(3) > a'),
                lambda: self.driver.find_element(By.XPATH, '//*[@id="ItauTables_Table_1"]/tbody/tr[1]/td[3]/a'),
                lambda: self.driver.find_element(By.XPATH, '/html/body/div[1]/section/div/div/section/div[6]/div[3]/div/div[2]/div[4]/div/div/table/tbody/tr[1]/td[3]/a'),
                lambda: WebDriverWait(self.driver, 10).until(EC.presence_of_element_located((By.XPATH, '//a[contains(@class, "linkAzul") and contains(@onclick, "BENEFICIARIO FINAL")]'))),
                lambda: WebDriverWait(self.driver, 10).until(EC.presence_of_element_located((By.XPATH, '//a[@onclick="validaBeneficiarioIgual(\'BENEFICIARIO FINAL\', \'CNPJ DO BENEFICIARIO\', \'26f76f39-4b35-4eb1-96d1-0b1a3f4bd5fe\');"]')))
            ]

            for locator in beneficiario_locators:
                try:
                    beneficiario_element = locator()
                    logging.info("Beneficiário encontrado!")
                    beneficiario_element.click()
                    break
                except NoSuchElementException:
                    logging.warning("Beneficiário não encontrado. Tentando o próximo localizador...")
        except Exception as e:
            logging.error("Erro ao selecionar beneficiário", exc_info=True)
            
    def confirmar_e_emitir_boleto(self):
        try:
            # Esperar o botão "continuar" ficar visível e clicável
            confirmar_locators = [
                (By.CSS_SELECTOR, "#continuar"),
                (By.XPATH, "//a[@id='continuar']"),
                (By.XPATH, "//a[contains(@class, 'itau-button')]"),
                (By.XPATH, "//a[text()='continuar']"),
                (By.XPATH, "//a[@role='button' and @id='continuar']"),
                (By.XPATH, "//a[@href='javascript:;' and contains(@class, 'cursorPointer') and contains(@class, 'itau-button')]"),
                (By.XPATH, "//a[@role='button' and contains(text(), 'continuar')]"),
                (By.XPATH, "//a[contains(@onclick, 'campos.continuar.click()')]"),
                (By.XPATH, "//div[@class='container col9']//a[@id='continuar']")
            ]

            actions = ActionChains(self.driver)
            sucesso_confirmar = False

            # Adicionar um tab para garantir que o foco saia do iframe/modal
            actions.send_keys(Keys.TAB).perform()
            time.sleep(1)

            for locator in confirmar_locators:
                try:
                    confirmar_button = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable(locator))
                    actions.move_to_element(confirmar_button).click().perform()
                    sucesso_confirmar = True
                    break
                except:
                    continue  

            if sucesso_confirmar:
                time.sleep(5)  # Aguarde um tempo para garantir que a próxima ação possa ser realizada
                emitir_boleto_button = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable((By.CSS_SELECTOR, "#efetivar")))
                emitir_boleto_button.click()
                logging.info("Boleto emitido - 1")
                time.sleep(5)

                # Localizadores para salvar o PDF
                salvar_pdf_locators = [
                    (By.CSS_SELECTOR, "#boletoEmitido > div > fieldset:nth-child(4) > div > div:nth-child(1) > a"),
                    (By.XPATH, "/html/body/div[1]/section/div/div[1]/section/section/div/div[1]/form[2]/div/fieldset[3]/div/div[1]/a"),
                    (By.XPATH, "//a[@role='button' and contains(@class, 'itau-button--outline') and contains(@class, 'itau-button')]"),
                    (By.XPATH, "//a[@role='button' and @tabindex='0' and contains(@class, 'itau-button--outline') and contains(@class, 'itau-button')]"),
                    (By.XPATH, "//a[contains(@class, 'btn-action-pdf') and contains(text(), 'salvar em PDF')]")
                ]

                sucesso_salvar = False
                for locator in salvar_pdf_locators:
                    try:
                        salvar_pdf_button = WebDriverWait(self.driver, 20).until(EC.element_to_be_clickable(locator))
                        salvar_pdf_button.click()
                        logging.info("Botão de salvar PDF clicado")
                        sucesso_salvar = True
                        break
                    except Exception as e:
                        logging.warning(f"Erro ao tentar clicar no botão de salvar PDF usando {locator}: {e}")
                        continue

                if not sucesso_salvar:
                    logging.error("Erro ao clicar no botão de salvar PDF")
                
                time.sleep(10)
            else:
                logging.error("Erro ao confirmar dados do boleto")
        except Exception as e:
            logging.error("Erro ao confirmar e emitir boleto", exc_info=True)


    def renomear_arquivo_baixado(self, dacte):
        try:
            diretorio_downloads = r"C:\Users\User\Downloads"
            for nome_arquivo in os.listdir(diretorio_downloads):
                if (nome_arquivo.startswith("QRCode_") or nome_arquivo.startswith("Boleto_")) and not any(char.isdigit() for char in nome_arquivo):
                    novo_nome_arquivo = f"Boleto_Dacte_{dacte}.pdf"
                    caminho_antigo = os.path.join(diretorio_downloads, nome_arquivo)
                    caminho_novo = os.path.join(diretorio_downloads, novo_nome_arquivo)
                    os.rename(caminho_antigo, caminho_novo)
                    time.sleep(5)
        except Exception as e:
            logging.error("Erro ao renomear arquivo baixado", exc_info=True)

    def fazer_boleto(self):
        try:
            self.ler_fatura()
            self.acessar_pagina_e_fazer_login()
            self.interagir_com_teclado_virtual()
            self.fechar_popups()

            dados_csv = self.ler_dados_csv()
            for linha in dados_csv:
                cnpj_cpf = linha['CPF/CNPJ']
                valor = linha['Valor']
                vencimento = linha['Data de vencimento']
                fatura = linha['Fatura']
                dacte = linha['Dacte']
                
                logging.info(f"Processando boleto para {cnpj_cpf} no valor de {valor}, vencimento {vencimento}, fatura {fatura}")
                time.sleep(10)
                
                self.navegar_no_menu()
                self.preencher_formulario_boleto(cnpj_cpf, valor, vencimento, fatura)
                self.selecionar_beneficiario()
                self.confirmar_e_emitir_boleto()
                self.renomear_arquivo_baixado(dacte)
        except Exception as e:
            logging.error("Erro ao processar boletos", exc_info=True)
        finally:
            self.driver.quit()
            for filename in os.listdir():
                if filename.endswith(".csv") and filename != "nomes.csv":
                    os.remove(filename)
          
    def fechar_popup_emitir_boleto(self):
        localizadores = [
            (By.ID, "fechar"),
            (By.XPATH, "/html/body/div[7]/div/div/div/div/div/div[2]/div[2]/a"),
            (By.CSS_SELECTOR, "#fechar"),
            (By.XPATH, "//a[@id='fechar']"),
            (By.XPATH, "//a[@role='button' and @data-lity-close]"),
            (By.XPATH, "//a[@href='javascript:;' and contains(@class, 'cursorPointer') and contains(@class, 'itau-button')]")
        ]
        
        for localizador in localizadores:
            try:
                fechar_button = WebDriverWait(self.driver, 10).until(EC.element_to_be_clickable(localizador))
                fechar_button.click()
                logging.info("Pop-up 'fechar' encontrado e clicado")
                return
            except NoSuchElementException:
                logging.info(f"Pop-up 'fechar' não encontrado usando {localizador}")
            except Exception as e:
                logging.error(f"Erro ao tentar fechar o pop-up 'fechar' usando {localizador}", exc_info=True)

        logging.info("Pop-up 'fechar' não encontrado com nenhum dos localizadores")

def close_special_popups(driver):
    try:
        driver.execute_script("window.top.closeModalPushNotification();")
        time.sleep(1)
        driver.execute_script("window.top._closeModalH2o('/21672839401/PJ_PopUp');")
        time.sleep(1)
        
        fechar_popup = driver.find_element(By.CSS_SELECTOR, 'body > div.GoogleActiveViewElement > button > span')
        fechar_popup.click()
        fechar_popup2 = driver.find_element(By.CSS_SELECTOR, 'body > div.GoogleActiveViewElement > button > span')
        fechar_popup2.click()
        
        logging.info("Anúncio fechado via JavaScript com sucesso.")
    except Exception as e:
        logging.error("Não foi possível fechar o anúncio via JavaScript", exc_info=True)

def close_popups(driver, attempts=3):
    for _ in range(attempts):
        try:
            close_special_popups(driver)
            driver.execute_script("""
                var closeButtons = document.querySelectorAll('[aria-label="close"], [aria-label="Close"], .close, .modal-close');
                closeButtons.forEach(button => {
                    if (button.offsetParent !== null) {
                        button.click();
                    }
                });

                var overlays = document.querySelectorAll('.overlay, .modal, .popup, .lightbox');
                overlays.forEach(overlay => {
                    if (overlay.offsetParent !== null) {
                        overlay.style.display = 'none';
                    }
                });

                var popups = document.querySelectorAll('div[style*="position: fixed"], div[style*="position: absolute"]');
                popups.forEach(popup => {
                    if ((popup.clientHeight > window.innerHeight * 0.5 || popup.clientWidth > window.innerWidth * 0.5) && popup.offsetParent !== null) {
                        popup.remove();
                    }
                });
            """)
            time.sleep(1)
        except Exception as e:
            logging.error("Erro ao tentar fechar pop-ups", exc_info=True)

