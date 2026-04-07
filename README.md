# SISTEMA-ADEGA
FIZ UM SISTEMAS DE VENDAS PARA UMA ADEGA COM BANCO DE DADOS PARA O ESTOQUE, TELA DE FATURAMENTO (ABERTO A SUJESTÕES DE ALTERAÇÃO)

import tkinter as tk
from tkinter import ttk, messagebox
import sqlite3
import os
import json
import shutil
from datetime import datetime, timedelta, date
from reportlab.lib.pagesizes import letter
from reportlab.pdfgen import canvas
from reportlab.lib import colors
from matplotlib.backends.backend_tkagg import FigureCanvasTkAgg # type: ignore
import matplotlib.pyplot as plt # type: ignore
import matplotlib.gridspec as gridspec

# =============================
# FORMATADOR PARA IMPRESSÃO TÉRMICA
# =============================

# Bibliotecas para impressora térmica (Fila de Impressão do Windows)
HAS_WIN32PRINT = False
try:
    import win32print
    import win32ui
    HAS_WIN32PRINT = True
except ImportError:
    HAS_WIN32PRINT = False

# =============================
# FORMATADOR PARA IMPRESSÃO TÉRMICA
# =============================
class ImpressoraTermica:
    LARGURA_BOBINA = 42 # Padrão para 80mm da TM-T20
    CONFIG_FILE = "config_impressora.json"

    @staticmethod
    def listar_impressoras_windows():
        impressoras = []
        if not HAS_WIN32PRINT:
            return impressoras
        try:
            # Listar todas as impressoras instaladas no Windows
            printers = win32print.EnumPrinters(win32print.PRINTER_ENUM_LOCAL | win32print.PRINTER_ENUM_CONNECTIONS)
            for p in printers:
                impressoras.append(p[2]) # O índice 2 é o nome da impressora
        except Exception as e:
            print(f"Erro ao listar impressoras: {e}")
        return impressoras

    @staticmethod
    def salvar_config(nome_impressora):
        try:
            config = {"impressora": nome_impressora}
            # Usa o caminho absoluto para evitar problemas com o executável
            caminho = os.path.join(os.path.dirname(os.path.abspath(__file__)), ImpressoraTermica.CONFIG_FILE)
            with open(caminho, "w", encoding="utf-8") as f:
                json.dump(config, f, indent=4)
            return True
        except Exception as e:
            print(f"Erro ao salvar config: {e}")
            return False

    @staticmethod
    def carregar_config():
        caminho = os.path.join(os.path.dirname(os.path.abspath(__file__)), ImpressoraTermica.CONFIG_FILE)
        if os.path.exists(caminho):
            try:
                with open(caminho, "r", encoding="utf-8") as f:
                    data = json.load(f)
                    if isinstance(data, dict) and data.get("impressora"):
                        return data
            except Exception as e:
                print(f"Erro ao carregar config: {e}")
        return {"impressora": ""} # Retorna vazio se não houver config válida

    @staticmethod
    def remover_acentos(texto):
        import unicodedata
        # Normaliza o texto para remover acentos e caracteres especiais que travam a térmica
        texto = unicodedata.normalize('NFD', str(texto))
        texto = texto.encode('ascii', 'ignore').decode('utf-8')
        return texto

    @staticmethod
    def imprimir_arquivo(caminho_arquivo):
        if not HAS_WIN32PRINT:
            messagebox.showerror("Erro", "Biblioteca 'pywin32' não instalada.\nInstale com: pip install pywin32")
            return False

        config = ImpressoraTermica.carregar_config()
        nome_impressora = config.get("impressora")

        if not nome_impressora or nome_impressora == "":
            messagebox.showwarning("Impressora não encontrada", 
                "Nenhuma impressora foi selecionada.\n\n"
                "Vá em 'Configurações', escolha a impressora na lista e clique em 'Salvar'.")
            return False

        try:
            # Tenta ler o arquivo com UTF-8, mas trata erros de decodificação
            try:
                with open(caminho_arquivo, "r", encoding="utf-8") as f:
                    conteudo = f.read()
            except UnicodeDecodeError:
                with open(caminho_arquivo, "r", encoding="latin-1") as f:
                    conteudo = f.read()

            # Limpa o texto de acentos para compatibilidade total com a Epson
            conteudo_limpo = ImpressoraTermica.remover_acentos(conteudo)

            # Comando para cortar papel na Epson (ESC/POS)
            comando_corte = "\x1b\x69"

            hPrinter = win32print.OpenPrinter(nome_impressora)
            try:
                # O modo "RAW" é necessário para enviar comandos ESC/POS como o corte
                hJob = win32print.StartDocPrinter(hPrinter, 1, ("Recibo Rota dos Drinks", None, "RAW"))
                try:
                    win32print.StartPagePrinter(hPrinter)
                    # Envia o conteúdo usando latin-1 que é o padrão de muitas térmicas
                    dados = (conteudo_limpo + comando_corte).encode("latin-1", "replace")
                    win32print.WritePrinter(hPrinter, dados)
                    win32print.EndPagePrinter(hPrinter)
                finally:
                    win32print.EndDocPrinter(hPrinter)
            finally:
                win32print.ClosePrinter(hPrinter)
            return True
        except Exception as e:
            messagebox.showerror("Erro de Impressão", f"Erro ao imprimir na '{nome_impressora}':\n{e}")
            return False

    @staticmethod
    def centralizar(texto):
        return texto.center(ImpressoraTermica.LARGURA_BOBINA)

    @staticmethod
    def formatar_linha(item, qtd, subtotal):
        nome = str(item)[:15].ljust(15)
        quantidade = str(qtd).center(6)
        valor = f"R${subtotal:>.2f}".rjust(10)
        return f"{nome}{quantidade}{valor}"

    @staticmethod
    def gerar_texto_venda(venda_id, data, total, pagamento, itens, cliente_nome=None):
        linhas = []
        linhas.append(ImpressoraTermica.centralizar("ROTA DOS DRINKS"))
        linhas.append(ImpressoraTermica.centralizar("--------------------------------"))
        linhas.append(f"Venda ID: {venda_id}")
        linhas.append(f"Data: {data}")
        if cliente_nome:
            linhas.append(f"Cliente: {cliente_nome[:20]}")
        linhas.append("--------------------------------")
        linhas.append("Item           Qtd      Subtotal")
        for item in itens:
            linhas.append(ImpressoraTermica.formatar_linha(item[0], item[1], item[3]))
        linhas.append("--------------------------------")
        linhas.append(f"TOTAL: R$ {total:.2f}".rjust(ImpressoraTermica.LARGURA_BOBINA))
        linhas.append(f"Pagamento: {pagamento}")

        # Adição de QR Code Pix se for Pix
        if pagamento.upper() == "PIX":
            linhas.append("\n" + ImpressoraTermica.centralizar("PAGUE COM PIX"))
            pix_copia_e_cola = f"00020126360014br.gov.bcb.pix0114SUACHAVEPIX0503Venda{venda_id}520400005303986540{total:.2f}5802BR5915ROTA DOS DRINKS6008SAO PAULO62070503***6304"
            linhas.append(pix_copia_e_cola[:ImpressoraTermica.LARGURA_BOBINA])
            linhas.append(pix_copia_e_cola[ImpressoraTermica.LARGURA_BOBINA:ImpressoraTermica.LARGURA_BOBINA*2])
            linhas.append(ImpressoraTermica.centralizar("(Escaneie o QR Code no Balcao)"))

        linhas.append("\n" + ImpressoraTermica.centralizar("OBRIGADO PELA PREFERENCIA!"))
        linhas.append("\n\n\n")

        conteudo = "\n".join(linhas)
        filename = f"termico_venda_{venda_id}.txt"
        with open(filename, "w") as f:
            f.write(conteudo)
        return filename, conteudo

    @staticmethod
    def gerar_texto_debitos(debitos_selecionados):
        linhas = []
        linhas.append(ImpressoraTermica.centralizar("ROTA DOS DRINKS"))
        linhas.append(ImpressoraTermica.centralizar("RELATORIO DE DEBITOS"))
        linhas.append(ImpressoraTermica.centralizar("--------------------------------"))
        linhas.append(f"Data: {datetime.now().strftime('%d/%m/%Y %H:%M')}")
        linhas.append("--------------------------------")
        linhas.append("ID   Cliente          Valor")

        total_pendente = 0
        for d in debitos_selecionados:
            id_d = str(d[0]).ljust(4)
            cliente = str(d[1])[:15].ljust(15)
            try:
                valor_num = float(d[2])
            except (ValueError, TypeError):
                valor_num = 0.0
            valor = f"R${valor_num:.2f}".rjust(10)
            linhas.append(f"{id_d}{cliente}{valor}")
            total_pendente += valor_num

        linhas.append("--------------------------------")
        linhas.append(f"TOTAL: R$ {total_pendente:.2f}".rjust(ImpressoraTermica.LARGURA_BOBINA))
        linhas.append("\n\n\n")

        conteudo = "\n".join(linhas)
        filename = f"termico_debitos_{date.today()}.txt"
        with open(filename, "w") as f:
            f.write(conteudo)
        return filename, conteudo

    @staticmethod
    def gerar_comanda_delivery_termica(pedido_id, data_hora, cliente_nome, cliente_tel, endereco, bairro, compl, itens, total, taxa, pagamento, troco_para=None, obs=None, motoboy=None):
        L = ImpressoraTermica.LARGURA_BOBINA
        linhas = []
        linhas.append("=" * L)
        linhas.append(ImpressoraTermica.centralizar("ROTA DOS DRINKS"))
        linhas.append(ImpressoraTermica.centralizar("** COMANDA DE DELIVERY **"))
        linhas.append("=" * L)
        linhas.append(f"Pedido No: {pedido_id}")
        linhas.append(f"Data/Hora: {data_hora}")
        linhas.append("-" * L)
        linhas.append(ImpressoraTermica.centralizar("[ DADOS DO CLIENTE ]"))
        linhas.append(f"Nome    : {str(cliente_nome)[:30]}")
        linhas.append(f"Fone    : {str(cliente_tel)[:20]}")
        linhas.append("-" * L)
        linhas.append(ImpressoraTermica.centralizar("[ ENDERECO DE ENTREGA ]"))
        linhas.append(f"Rua/Av  : {str(endereco)[:30]}")
        linhas.append(f"Bairro  : {str(bairro)[:25]}")
        if compl: linhas.append(f"Compl.  : {str(compl)[:28]}")
        linhas.append("-" * L)
        linhas.append(ImpressoraTermica.centralizar("[ ITENS DO PEDIDO ]"))
        linhas.append(f"{'Produto':<18}{'Qtd':^5}{'Subtotal':>{L-23}}")
        linhas.append("-" * L)
        for item in itens:
            nome_fmt = str(item[0])[:18].ljust(18)
            qtd_fmt  = str(item[1]).center(5)
            val_fmt  = f"R${item[3]:.2f}".rjust(L - 23)
            linhas.append(f"{nome_fmt}{qtd_fmt}{val_fmt}")
        linhas.append("-" * L)
        if obs:
            linhas.append(f"OBS: {obs[:L-5]}")
            linhas.append("-" * L)
        total_geral = total + taxa
        linhas.append(f"Subtotal Produtos : R$ {total:>7.2f}".rjust(L))
        linhas.append(f"Taxa de Entrega   : R$ {taxa:>7.2f}".rjust(L))
        linhas.append("=" * L)
        linhas.append(f"TOTAL A PAGAR     : R$ {total_geral:>7.2f}".rjust(L))
        linhas.append("=" * L)
        linhas.append(f"Pagamento : {pagamento}")

        # QR Code Pix no Delivery
        if "PIX" in pagamento.upper():
            linhas.append("-" * L)
            linhas.append(ImpressoraTermica.centralizar("PAGAMENTO VIA PIX"))
            pix_code = f"00020126360014br.gov.bcb.pix0114SUACHAVEPIX0503Del{pedido_id}520400005303986540{total_geral:.2f}5802BR5915ROTA DOS DRINKS6008SAO PAULO62070503***6304"
            # Formata o código Pix Copia e Cola em blocos para caber na térmica
            for i in range(0, len(pix_code), L):
                linhas.append(pix_code[i:i+L])
            linhas.append("-" * L)

        if troco_para and any(p in pagamento.lower() for p in ["dinheiro", "especie", "espécie"]):
            linhas.append(f"Troco p/ : R$ {troco_para:.2f}  =>  Troco: R$ {troco_para-total_geral:.2f}")

        linhas.append("-" * L)
        linhas.append(ImpressoraTermica.centralizar("[ MOTOBOY ]"))
        linhas.append(f"Entregador: {str(motoboy or '_________________')[:28]}")
        linhas.append("\nAssinatura do Cliente:\n\n" + ("_" * (L-2)) + "\n" + ("-" * L))
        linhas.append(ImpressoraTermica.centralizar("OBRIGADO PELA PREFERENCIA!"))
        linhas.append("\n\n\n")

        conteudo = "\n".join(linhas)
        filename = f"delivery_{pedido_id}.txt"
        with open(filename, "w") as f:
            f.write(conteudo)
        return filename, conteudo

# =============================
# GERADOR DE RELATÓRIOS PDF
# =============================
class GeradorPDF:
    @staticmethod
    def gerar_recibo_venda(venda_id, data, total, pagamento, itens, cliente_nome=None):
        filename = f"recibo_venda_{venda_id}.pdf"
        c = canvas.Canvas(filename, pagesize=(300, 600))
        c.setFont("Helvetica-Bold", 14)
        c.drawString(10, 580, "ROTA DOS DRINKS")
        c.setFont("Helvetica", 10)
        c.drawString(10, 565, f"Data: {data}")
        c.drawString(10, 550, f"Venda ID: {venda_id}")
        if cliente_nome:
            c.drawString(10, 535, f"Cliente: {cliente_nome}")

        c.line(10, 530, 290, 530)
        c.drawString(10, 515, "Item")
        c.drawString(150, 515, "Qtd")
        c.drawString(200, 515, "V. Unit")
        c.drawString(250, 515, "Sub")

        y = 500
        for item in itens:
            # item = (nome, qtd, preco, subtotal)
            c.drawString(10, y, str(item[0])[:20])
            c.drawString(150, y, str(item[1]))
            c.drawString(200, y, f"{item[2]:.2f}")
            c.drawString(250, y, f"{item[3]:.2f}")
            y -= 15
            if y < 50: break

        c.line(10, y, 290, y)
        y -= 20
        c.setFont("Helvetica-Bold", 12)
        c.drawString(10, y, f"TOTAL: R$ {total:.2f}")
        y -= 15
        c.setFont("Helvetica", 10)
        c.drawString(10, y, f"Pagamento: {pagamento}")
        c.save()
        return filename

    @staticmethod
    def gerar_relatorio_estoque(produtos):
        filename = f"relatorio_estoque_{date.today()}.pdf"
        c = canvas.Canvas(filename, pagesize=letter)
        width, height = letter
        c.setFont("Helvetica-Bold", 16)
        c.drawString(50, height - 50, "RELATÓRIO DE ESTOQUE - ROTA DOS DRINKS")
        c.setFont("Helvetica", 10)
        c.drawString(50, height - 70, f"Data de Emissão: {datetime.now().strftime('%d/%m/%Y %H:%M')}")

        c.line(50, height - 80, width - 50, height - 80)
        c.drawString(50, height - 95, "ID")
        c.drawString(100, height - 95, "Produto")
        c.drawString(350, height - 95, "Preço Venda")
        c.drawString(450, height - 95, "Estoque")

        y = height - 110
        for p in produtos:
            # p = (id, nome, preco, estoque)
            c.drawString(50, y, str(p[0]))
            c.drawString(100, y, str(p[1]))
            c.drawString(350, y, f"R$ {p[2]:.2f}")
            c.drawString(450, y, str(p[3]))
            y -= 15
            if y < 50:
                c.showPage()
                y = height - 50
        c.save()
        return filename

    @staticmethod
    def gerar_relatorio_debitos(debitos):
        filename = f"relatorio_debitos_{date.today()}.pdf"
        c = canvas.Canvas(filename, pagesize=letter)
        width, height = letter
        c.setFont("Helvetica-Bold", 16)
        c.drawString(50, height - 50, "RELATÓRIO DE DÉBITOS (FIADO) - ROTA DOS DRINKS")
        c.setFont("Helvetica", 10)
        c.drawString(50, height - 70, f"Data de Emissão: {datetime.now().strftime('%d/%m/%Y %H:%M')}")
        c.line(50, height - 80, width - 50, height - 80)
        c.drawString(50, height - 95, "ID")
        c.drawString(100, height - 95, "Cliente")
        c.drawString(300, height - 95, "Valor")
        c.drawString(400, height - 95, "Data")
        y = height - 110
        total_pendente = 0
        for d in debitos:
            c.drawString(50, y, str(d[0]))
            c.drawString(100, y, str(d[1]))
            c.drawString(300, y, f"R$ {d[2]:.2f}")
            c.drawString(400, y, str(d[3]))
            total_pendente += d[2]
            y -= 15
            if y < 50:
                c.showPage()
                y = height - 50
        c.line(50, y, width - 50, y)
        y -= 20
        c.setFont("Helvetica-Bold", 12)
        c.drawString(50, y, f"TOTAL PENDENTE: R$ {total_pendente:.2f}")
        c.save()
        return filename

# =============================
# CONFIGURAÇÕES GLOBAIS
# =============================
USUARIO_LOGADO = None
CARGO_LOGADO = None
LIMITE_ESTOQUE_BAIXO = 5

# =============================
# TELA DE LOGIN
# =============================
class LoginApp:
    def __init__(self, master, on_login_success):
        self.master = master
        self.on_login_success = on_login_success
        master.title("ROTA DOS DRINKS - Acesso")
        master.geometry("450x500")
        master.resizable(False, False)
        master.configure(bg="#F0F0F0")

        # Estilo para o Notebook
        style = ttk.Style()
        style.configure("TNotebook", background="#F0F0F0")
        style.configure("TNotebook.Tab", font=("Segoe UI", 10, "bold"), padding=[10, 5])

        self.notebook = ttk.Notebook(master)
        self.notebook.pack(expand=True, fill="both", padx=10, pady=10)

        # ABA LOGIN
        self.tab_login = tk.Frame(self.notebook, bg="white", padx=20, pady=20)
        self.notebook.add(self.tab_login, text="  ENTRAR  ")

        tk.Label(self.tab_login, text="ROTA DOS DRINKS", font=("Segoe UI", 18, "bold"), bg="white", fg="#333").pack(pady=(0, 20))

        tk.Label(self.tab_login, text="Usuário:", bg="white", font=("Segoe UI", 10)).pack(anchor="w", pady=(5, 0))
        self.entry_username = tk.Entry(self.tab_login, font=("Segoe UI", 11))
        self.entry_username.pack(fill="x", pady=5)

        tk.Label(self.tab_login, text="Senha:", bg="white", font=("Segoe UI", 10)).pack(anchor="w", pady=(5, 0))
        self.entry_password = tk.Entry(self.tab_login, show="*", font=("Segoe UI", 11))
        self.entry_password.pack(fill="x", pady=5)

        self.button_login = tk.Button(self.tab_login, text="ACESSAR SISTEMA", command=self.login, bg="#4CAF50", fg="white", font=("Segoe UI", 11, "bold"), pady=8, cursor="hand2")
        self.button_login.pack(fill="x", pady=25)

        tk.Label(self.tab_login, text="Acesso restrito a pessoal autorizado", font=("Segoe UI", 8), bg="white", fg="gray").pack()

        # ABA CADASTRO
        self.tab_register = tk.Frame(self.notebook, bg="white", padx=20, pady=20)
        self.notebook.add(self.tab_register, text="  CADASTRAR  ")

        tk.Label(self.tab_register, text="NOVO USUÁRIO", font=("Segoe UI", 16, "bold"), bg="white", fg="#333").pack(pady=(0, 15))

        tk.Label(self.tab_register, text="Nome de Usuário:", bg="white", font=("Segoe UI", 10)).pack(anchor="w", pady=(5, 0))
        self.reg_user = tk.Entry(self.tab_register, font=("Segoe UI", 11))
        self.reg_user.pack(fill="x", pady=5)

        tk.Label(self.tab_register, text="Senha (mín. 6 caracteres):", bg="white", font=("Segoe UI", 10)).pack(anchor="w", pady=(5, 0))
        self.reg_pass = tk.Entry(self.tab_register, show="*", font=("Segoe UI", 11))
        self.reg_pass.pack(fill="x", pady=5)

        tk.Label(self.tab_register, text="Cargo / Nível de Acesso:", bg="white", font=("Segoe UI", 10)).pack(anchor="w", pady=(5, 0))
        self.reg_role = ttk.Combobox(self.tab_register, values=["funcionario", "admin"], font=("Segoe UI", 10), state="readonly")
        self.reg_role.set("funcionario")
        self.reg_role.pack(fill="x", pady=5)

        self.button_reg = tk.Button(self.tab_register, text="CRIAR CONTA", command=self.registrar, bg="#2196F3", fg="white", font=("Segoe UI", 11, "bold"), pady=8, cursor="hand2")
        self.button_reg.pack(fill="x", pady=25)

        tk.Label(self.tab_register, text="* Administradores têm acesso total ao sistema", font=("Segoe UI", 8), bg="white", fg="gray").pack()

        criar_tabela_usuarios()

    def login(self):
        username = self.entry_username.get().strip()
        password = self.entry_password.get().strip()

        if not username or not password:
            messagebox.showwarning("Aviso", "Preencha todos os campos.")
            return

        if len(password) < 6:
            messagebox.showwarning("Aviso de Segurança", "A senha deve ter no mínimo 6 caracteres.")
            return

        conn = sqlite3.connect("sistema.db")
        cursor = conn.cursor()
        cursor.execute("SELECT username, role FROM usuarios WHERE username = ? AND password = ?", (username, password))
        user = cursor.fetchone()
        conn.close()

        if user:
            global USUARIO_LOGADO, CARGO_LOGADO
            USUARIO_LOGADO = user[0]
            CARGO_LOGADO = user[1]
            messagebox.showinfo("Login", f"Bem-vindo, {USUARIO_LOGADO} ({CARGO_LOGADO})!")
            self.master.withdraw()
            self.on_login_success()
        else:
            messagebox.showerror("Login", "Usuário ou senha inválidos.")

    def registrar(self):
        u, p, r = self.reg_user.get().strip(), self.reg_pass.get().strip(), self.reg_role.get()
        if not u or not p:
            messagebox.showerror("Erro", "Preencha todos os campos")
            return
        if len(p) < 6:
            messagebox.showerror("Erro", "A senha deve ter no mínimo 6 caracteres")
            return

        resultado = cadastrar_usuario(u, p, r)
        if resultado == True:
            messagebox.showinfo("Sucesso", f"Usuário {u} ({r}) cadastrado com sucesso!")
            self.reg_user.delete(0, tk.END)
            self.reg_pass.delete(0, tk.END)
            self.notebook.select(0) # Volta para a aba de login
        elif resultado == "existe":
            messagebox.showerror("Erro", f"O nome de usuário '{u}' já está em uso. Escolha outro.")
        elif resultado == "limite_admin":
            messagebox.showerror("Erro", "Limite de 3 administradores atingido. Não é possível cadastrar mais administradores.")
        else:
            messagebox.showerror("Erro", f"Erro ao cadastrar: {resultado}")

dashboard_ativo = False

# =============================
# BANCO DE DADOS
# =============================

def criar_tabela_usuarios():
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS usuarios (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        username TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        role TEXT NOT NULL DEFAULT 'funcionario'
    )""")
    conn.commit()
    conn.close()

def cadastrar_usuario(username, password, role='funcionario'):
    try:
        conn = sqlite3.connect("sistema.db")
        cursor = conn.cursor()
        # Verificar explicitamente se o usuário existe antes de tentar inserir
        cursor.execute("SELECT id FROM usuarios WHERE username = ?", (username,))
        if cursor.fetchone():
            conn.close()
            return "existe"

        if role == 'admin':
            cursor.execute("SELECT COUNT(*) FROM usuarios WHERE role = 'admin'")
            if cursor.fetchone()[0] >= 3:
                conn.close()
                return "limite_admin"

        cursor.execute("INSERT INTO usuarios (username, password, role) VALUES (?, ?, ?)", (username, password, role))
        conn.commit()
        conn.close()
        return True
    except Exception as e:
        return str(e)

def criar_banco():
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    cursor.execute("CREATE TABLE IF NOT EXISTS produtos (id INTEGER PRIMARY KEY AUTOINCREMENT, codigo_barras TEXT UNIQUE, nome TEXT UNIQUE, preco REAL, estoque INTEGER)")
    cursor.execute("CREATE TABLE IF NOT EXISTS vendas (id INTEGER PRIMARY KEY AUTOINCREMENT, data TEXT, total REAL, pagamento TEXT)")
    cursor.execute("CREATE TABLE IF NOT EXISTS itens_venda (id INTEGER PRIMARY KEY AUTOINCREMENT, venda_id INTEGER, produto_id INTEGER, nome_produto TEXT, quantidade INTEGER, preco_unitario REAL, subtotal REAL)")

    try:
        cursor.execute("ALTER TABLE produtos ADD COLUMN preco_custo REAL DEFAULT 0.0")
    except sqlite3.OperationalError:
        pass

    try:
        cursor.execute("ALTER TABLE itens_venda ADD COLUMN preco_custo_unitario REAL DEFAULT 0.0")
    except sqlite3.OperationalError:
        pass

    try:
        cursor.execute("ALTER TABLE vendas ADD COLUMN cliente_id INTEGER")
    except sqlite3.OperationalError:
        pass
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS clientes (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        nome TEXT UNIQUE NOT NULL,
        telefone TEXT,
        limite_fiado REAL DEFAULT 1000.0
    )""")
    cursor.execute("""
    CREATE TABLE IF NOT EXISTS debitos (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        cliente_id INTEGER,
        venda_id INTEGER,
        valor REAL,
        status TEXT DEFAULT 'PENDENTE',
        data TEXT,
        FOREIGN KEY(cliente_id) REFERENCES clientes(id),
        FOREIGN KEY(venda_id) REFERENCES vendas(id)
    )""")
    conn.commit()
    conn.close()

criar_banco()

# =============================
# 📊 FUNÇÕES DE FATURAMENTO E LUCRO
# =============================

def obter_metricas_faturamento():
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    hoje = date.today()
    hoje_str      = hoje.strftime("%Y-%m-%d")
    ontem_str     = (hoje - timedelta(days=1)).strftime("%Y-%m-%d")

    cursor.execute("""
        SELECT COALESCE(SUM(v.total),0), COUNT(DISTINCT v.id), 
               COALESCE(SUM(i.quantidade * i.preco_custo_unitario), 0)
        FROM vendas v
        LEFT JOIN itens_venda i ON v.id = i.venda_id
        WHERE substr(v.data,1,10)=?
    """, (hoje_str,))
    fat_hoje, nv_hoje, custo_hoje = cursor.fetchone()
    lucro_hoje = fat_hoje - custo_hoje

    cursor.execute("""
        SELECT COALESCE(SUM(v.total),0), COUNT(DISTINCT v.id),
               COALESCE(SUM(i.quantidade * i.preco_custo_unitario), 0)
        FROM vendas v
        LEFT JOIN itens_venda i ON v.id = i.venda_id
        WHERE substr(v.data,1,10)=?
    """, (ontem_str,))
    fat_ontem, nv_ontem, custo_ontem = cursor.fetchone()
    lucro_ontem = fat_ontem - custo_ontem

    seg_atual = hoje - timedelta(days=hoje.weekday())
    seg_ant   = seg_atual - timedelta(weeks=1)
    dom_ant   = seg_atual - timedelta(days=1)

    cursor.execute("""
        SELECT COALESCE(SUM(v.total),0), COUNT(DISTINCT v.id),
               COALESCE(SUM(i.quantidade * i.preco_custo_unitario), 0)
        FROM vendas v
        LEFT JOIN itens_venda i ON v.id = i.venda_id
        WHERE substr(v.data,1,10)>=? AND substr(v.data,1,10)<=?
    """, (seg_atual.strftime("%Y-%m-%d"), hoje_str))
    fat_semana, nv_semana, custo_semana = cursor.fetchone()
    lucro_semana = fat_semana - custo_semana

    cursor.execute("""
        SELECT COALESCE(SUM(v.total),0), COUNT(DISTINCT v.id),
               COALESCE(SUM(i.quantidade * i.preco_custo_unitario), 0)
        FROM vendas v
        LEFT JOIN itens_venda i ON v.id = i.venda_id
        WHERE substr(v.data,1,10)>=? AND substr(v.data,1,10)<=?
    """, (seg_ant.strftime("%Y-%m-%d"), dom_ant.strftime("%Y-%m-%d")))
    fat_sem_ant, nv_sem_ant, custo_sem_ant = cursor.fetchone()
    lucro_sem_ant = fat_sem_ant - custo_sem_ant

    mes_ini   = hoje.replace(day=1).strftime("%Y-%m-%d")
    if hoje.month == 1:
        mes_ant_ini = hoje.replace(year=hoje.year-1, month=12, day=1).strftime("%Y-%m-%d")
        mes_ant_fim = hoje.replace(year=hoje.year-1, month=12, day=31).strftime("%Y-%m-%d")
    else:
        mes_ant_ini = hoje.replace(month=hoje.month-1, day=1).strftime("%Y-%m-%d")
        ultimo_dia_ant = hoje.replace(day=1) - timedelta(days=1)
        mes_ant_fim = ultimo_dia_ant.strftime("%Y-%m-%d")

    cursor.execute("""
        SELECT COALESCE(SUM(v.total),0), COUNT(DISTINCT v.id),
               COALESCE(SUM(i.quantidade * i.preco_custo_unitario), 0)
        FROM vendas v
        LEFT JOIN itens_venda i ON v.id = i.venda_id
        WHERE substr(v.data,1,10)>=? AND substr(v.data,1,10)<=?
    """, (mes_ini, hoje_str))
    fat_mes, nv_mes, custo_mes = cursor.fetchone()
    lucro_mes = fat_mes - custo_mes

    cursor.execute("""
        SELECT COALESCE(SUM(v.total),0), COUNT(DISTINCT v.id),
               COALESCE(SUM(i.quantidade * i.preco_custo_unitario), 0)
        FROM vendas v
        LEFT JOIN itens_venda i ON v.id = i.venda_id
        WHERE substr(v.data,1,10)>=? AND substr(v.data,1,10)<=?
    """, (mes_ant_ini, mes_ant_fim))
    fat_mes_ant, nv_mes_ant, custo_mes_ant = cursor.fetchone()
    lucro_mes_ant = fat_mes_ant - custo_mes_ant

    conn.close()

    def ticket(fat, nv):
        return fat / nv if nv > 0 else 0.0

    def variacao(atual, anterior):
        if anterior == 0: return None
        return ((atual - anterior) / anterior) * 100

    def margem(lucro, fat):
        return (lucro / fat * 100) if fat > 0 else 0.0

    return {
        "diario":  {"fat": fat_hoje,  "lucro": lucro_hoje, "margem": margem(lucro_hoje, fat_hoje), "nv": nv_hoje,  "ticket": ticket(fat_hoje,  nv_hoje),
                    "fat_ant": fat_ontem,  "nv_ant": nv_ontem,  "var": variacao(fat_hoje, fat_ontem),
                    "label_ant": "vs ontem"},
        "semanal": {"fat": fat_semana, "lucro": lucro_semana, "margem": margem(lucro_semana, fat_semana), "nv": nv_semana,"ticket": ticket(fat_semana, nv_semana),
                    "fat_ant": fat_sem_ant,"nv_ant": nv_sem_ant,"var": variacao(fat_semana, fat_sem_ant),
                    "label_ant": "vs semana anterior"},
        "mensal":  {"fat": fat_mes, "lucro": lucro_mes, "margem": margem(lucro_mes, fat_mes), "nv": nv_mes,   "ticket": ticket(fat_mes,   nv_mes),
                    "fat_ant": fat_mes_ant,"nv_ant": nv_mes_ant,"var": variacao(fat_mes, fat_mes_ant),
                    "label_ant": "vs mês anterior"},
    }

def obter_serie_diaria(dias=30):
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    inicio = (date.today() - timedelta(days=dias-1)).strftime("%Y-%m-%d")
    cursor.execute("""
        SELECT substr(data,1,10) AS d, COALESCE(SUM(total),0)
        FROM vendas WHERE substr(data,1,10) >= ?
        GROUP BY d ORDER BY d
    """, (inicio,))
    rows = cursor.fetchall()
    conn.close()
    mapa = {r[0]: r[1] for r in rows}
    datas, totais = [], []
    for i in range(dias):
        d = (date.today() - timedelta(days=dias-1-i)).strftime("%Y-%m-%d")
        datas.append(d[5:])
        totais.append(mapa.get(d, 0))
    return datas, totais

def obter_serie_semanal(semanas=8):
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    labels, totais = [], []
    hoje = date.today()
    seg_atual = hoje - timedelta(days=hoje.weekday())
    for i in range(semanas-1, -1, -1):
        seg = seg_atual - timedelta(weeks=i)
        dom = seg + timedelta(days=6)
        cursor.execute("SELECT COALESCE(SUM(total),0) FROM vendas WHERE substr(data,1,10)>=? AND substr(data,1,10)<=?",
                       (seg.strftime("%Y-%m-%d"), dom.strftime("%Y-%m-%d")))
        totais.append(cursor.fetchone()[0])
        labels.append(f"{seg.strftime('%d/%m')}")
    conn.close()
    return labels, totais

def obter_serie_mensal(meses=12):
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    labels, totais = [], []
    hoje = date.today()
    for i in range(meses-1, -1, -1):
        mes_alvo = hoje.month - i
        ano_alvo = hoje.year
        while mes_alvo <= 0:
            mes_alvo += 12
            ano_alvo -= 1
        ini = date(ano_alvo, mes_alvo, 1).strftime("%Y-%m-%d")
        if mes_alvo == 12:
            fim = date(ano_alvo+1, 1, 1) - timedelta(days=1)
        else:
            fim = date(ano_alvo, mes_alvo+1, 1) - timedelta(days=1)
        fim_str = fim.strftime("%Y-%m-%d")
        cursor.execute("SELECT COALESCE(SUM(total),0) FROM vendas WHERE substr(data,1,10)>=? AND substr(data,1,10)<=?",
                       (ini, fim_str))
        totais.append(cursor.fetchone()[0])
        labels.append(f"{mes_alvo:02d}/{str(ano_alvo)[2:]}")
    conn.close()
    return labels, totais

# =============================
# FUNÇÕES DE CLIENTE
# =============================

def cadastrar_cliente():
    nome = entry_nome_cli.get().strip()
    tel  = entry_tel_cli.get().strip()
    limite = entry_limite_cli.get().strip()
    if not nome:
        messagebox.showerror("Erro", "O nome do cliente é obrigatório"); return
    try:
        limite = float(limite) if limite else 1000.0
    except:
        messagebox.showerror("Erro", "Limite inválido"); return
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    try:
        cursor.execute("INSERT INTO clientes (nome, telefone, limite_fiado) VALUES (?, ?, ?)", (nome, tel, limite))
        conn.commit()
        messagebox.showinfo("Sucesso", "Cliente cadastrado!")
        entry_nome_cli.delete(0, tk.END); entry_tel_cli.delete(0, tk.END); entry_limite_cli.delete(0, tk.END)
    except:
        messagebox.showerror("Erro", "Cliente já cadastrado")
    conn.close()

def editar_cliente_selecionado():
    selecionados = [tree_cli_list.item(i)["values"] for i in tree_cli_list.selection()]
    if not selecionados:
        messagebox.showerror("Erro", "Selecione um cliente para editar"); return
    cli = selecionados[0]
    id_cli, nome_cli, tel_cli, limite_cli = cli

    janela = tk.Toplevel(root); janela.title("Editar Cliente"); janela.geometry("350x250")
    tk.Label(janela, text="Nome:").pack(pady=2)
    ent_nome = tk.Entry(janela); ent_nome.insert(0, nome_cli); ent_nome.pack(pady=2)
    tk.Label(janela, text="Telefone:").pack(pady=2)
    ent_tel = tk.Entry(janela); ent_tel.insert(0, tel_cli); ent_tel.pack(pady=2)
    tk.Label(janela, text="Limite Fiado:").pack(pady=2)
    ent_limite = tk.Entry(janela); ent_limite.insert(0, limite_cli); ent_limite.pack(pady=2)

    def salvar():
        nome, tel, limite = ent_nome.get().strip(), ent_tel.get().strip(), ent_limite.get().strip()
        if not nome: messagebox.showerror("Erro", "Nome obrigatório"); return
        try: limite = float(limite)
        except: messagebox.showerror("Erro", "Limite inválido"); return
        conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
        cursor.execute("UPDATE clientes SET nome=?, telefone=?, limite_fiado=? WHERE id=?", (nome, tel, limite, id_cli))
        conn.commit(); conn.close(); janela.destroy(); carregar_clientes_lista()
        messagebox.showinfo("Sucesso", "Cliente atualizado!")

    tk.Button(janela, text="Salvar Alterações", command=salvar).pack(pady=10)

def remover_cliente_selecionado():
    selecionados = [tree_cli_list.item(i)["values"] for i in tree_cli_list.selection()]
    if not selecionados:
        messagebox.showerror("Erro", "Selecione um cliente para remover"); return
    cli = selecionados[0]
    if messagebox.askyesno("Confirmação", f"Deseja realmente remover the cliente {cli[1]}?"):
        conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
        cursor.execute("DELETE FROM clientes WHERE id=?", (cli[0],))
        conn.commit(); conn.close(); carregar_clientes_lista()
        messagebox.showinfo("Sucesso", "Cliente removido!")

def carregar_clientes_lista():
    tree_cli_list.delete(*tree_cli_list.get_children())
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT id, nome, telefone, limite_fiado FROM clientes")
    for r in cursor.fetchall(): tree_cli_list.insert("", "end", values=r)
    conn.close()

def buscar_cliente_fiado():
    janela = tk.Toplevel(root); janela.title("Selecionar Cliente"); janela.geometry("400x400")
    ttk.Label(janela, text="Selecione o cliente para a venda FIADO:").pack(pady=10)
    tree_cli = ttk.Treeview(janela, columns=("ID", "Nome", "Limite"), show="headings")
    for col in ["ID", "Nome", "Limite"]: tree_cli.heading(col, text=col)
    tree_cli.pack(pady=10, padx=10, fill="both", expand=True)
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT id, nome, limite_fiado FROM clientes"); rows = cursor.fetchall(); conn.close()
    for r in rows: tree_cli.insert("", "end", values=r)
    def confirmar():
        sel = tree_cli.selection()
        if not sel: messagebox.showerror("Erro", "Selecione um cliente"); return
        cli = tree_cli.item(sel[0])["values"]
        janela.destroy(); executar_finalizacao_venda(cli[0], cli[1])
    ttk.Button(janela, text="Confirmar Venda", command=confirmar).pack(pady=10)

# =============================
# FUNÇÕES DE PRODUTO E VENDA
# =============================

def gerar_codigo_interno():
    """Gera um código interno sequencial para produtos sem código de barras."""
    conn = sqlite3.connect("sistema.db")
    cursor = conn.cursor()
    # Busca o maior ID para gerar um código baseado nele
    cursor.execute("SELECT MAX(id) FROM produtos")
    max_id = cursor.fetchone()[0]
    conn.close()
    
    proximo_id = (max_id + 1) if max_id else 1
    # Formato INT-0001, INT-0002, etc.
    codigo = f"INT-{proximo_id:04d}"
    
    # Insere no campo de código de barras
    entry_cod_barras.delete(0, tk.END)
    entry_cod_barras.insert(0, codigo)

def cadastrar_produto():
    codigo_barras, nome, preco, custo, estoque = (entry_cod_barras.get().strip(), entry_nome_prod.get().strip(),
                                           entry_preco.get().strip(), entry_custo.get().strip(), entry_estoque.get().strip())
    if not codigo_barras or not nome or not preco or not custo or not estoque:
        messagebox.showerror("Erro", "Preencha todos os campos"); return
    try:
        preco, custo, estoque = float(preco), float(custo), int(estoque)
    except:
        messagebox.showerror("Erro", "Preço, custo ou estoque inválido"); return
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    try:
        cursor.execute("INSERT INTO produtos (codigo_barras, nome, preco, preco_custo, estoque) VALUES (?, ?, ?, ?, ?)",
                       (codigo_barras, nome, preco, custo, estoque))
        conn.commit()
        messagebox.showinfo("Sucesso", "Produto cadastrado!")
    except:
        messagebox.showerror("Erro", "Produto ou código já existe")
    conn.close()
    entry_cod_barras.delete(0, tk.END); entry_nome_prod.delete(0, tk.END)
    entry_preco.delete(0, tk.END); entry_custo.delete(0, tk.END); entry_estoque.delete(0, tk.END)
    atualizar_badge_alerta()

def editar_produto_selecionado():
    selecionados = [tree_est.item(i)["values"] for i in tree_est.get_children() if tree_est.item(i)["values"][0] == "☑"]
    if not selecionados:
        messagebox.showerror("Erro", "Selecione um produto (marcando o [X]) para editar"); return
    prod = selecionados[0]
    id_p, nome_p, preco_v, estoque_p, preco_c = prod[1], prod[2], prod[3], prod[4], prod[5]

    janela = tk.Toplevel(root); janela.title("Editar Produto"); janela.geometry("350x300")
    tk.Label(janela, text="Nome:").pack(pady=2)
    ent_nome = tk.Entry(janela); ent_nome.insert(0, nome_p); ent_nome.pack(pady=2)
    tk.Label(janela, text="Preço Venda:").pack(pady=2)
    ent_preco_v = tk.Entry(janela); ent_preco_v.insert(0, preco_v); ent_preco_v.pack(pady=2)
    tk.Label(janela, text="Preço Custo:").pack(pady=2)
    ent_preco_c = tk.Entry(janela); ent_preco_c.insert(0, preco_c); ent_preco_c.pack(pady=2)
    tk.Label(janela, text="Estoque:").pack(pady=2)
    ent_est = tk.Entry(janela); ent_est.insert(0, estoque_p); ent_est.pack(pady=2)

    def salvar():
        nome, pv, pc, est = ent_nome.get().strip(), ent_preco_v.get().strip(), ent_preco_c.get().strip(), ent_est.get().strip()
        if not nome: messagebox.showerror("Erro", "Nome obrigatório"); return
        try: pv, pc, est = float(pv), float(pc), int(est)
        except: messagebox.showerror("Erro", "Valores inválidos")
        conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
        cursor.execute("UPDATE produtos SET nome=?, preco=?, preco_custo=?, estoque=? WHERE id=?", (nome, pv, pc, est, id_p))
        conn.commit(); conn.close(); janela.destroy(); carregar_estoque()
        messagebox.showinfo("Sucesso", "Produto atualizado!")

    tk.Button(janela, text="Salvar Alterações", command=salvar).pack(pady=10)

def adicionar_produto_venda(codigo=None, quantidade=None):
    if not codigo: codigo = entry_codigo.get().strip()
    if not quantidade: 
        quantidade = entry_quantidade.get().strip()
        if not quantidade: quantidade = "1"

    if not str(quantidade).isdigit():
        messagebox.showerror("Erro", "Quantidade inválida"); return
    quantidade = int(quantidade)

    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    # Busca por código de barras OU nome exato
    cursor.execute("SELECT id, nome, preco, preco_custo, estoque FROM produtos WHERE codigo_barras = ? OR nome = ?", (codigo, codigo))
    produto = cursor.fetchone()

    if not produto:
        messagebox.showerror("Erro", "Produto não encontrado"); conn.close(); return

    id_produto, nome_produto, preco, custo, estoque = produto
    if quantidade > estoque:
        messagebox.showerror("Erro", "Estoque insuficiente"); conn.close(); return

    subtotal = preco * quantidade
    tree.insert("", "end", values=("☐", id_produto, nome_produto, quantidade, preco, subtotal, custo))
    atualizar_total()
    conn.close()
    entry_codigo.delete(0, tk.END); entry_quantidade.delete(0, tk.END)
    if hasattr(root, 'listbox_busca'): root.listbox_busca.place_forget()

def atualizar_total():
    total = sum(float(tree.item(i)["values"][5]) for i in tree.get_children())
    label_total.config(text=f"Total: R$ {total:.2f}")

# =============================
# FUNÇÕES DE DELIVERY
# =============================
def abrir_tela_delivery(itens, total, pagamento):
    janela = tk.Toplevel(root); janela.title("Dados de Delivery"); janela.geometry("400x550")
    tk.Label(janela, text="NOME CLIENTE:").pack(pady=2); ent_nome = ttk.Entry(janela); ent_nome.pack(fill="x", padx=20)
    tk.Label(janela, text="TELEFONE:").pack(pady=2); ent_tel = ttk.Entry(janela); ent_tel.pack(fill="x", padx=20)
    tk.Label(janela, text="ENDEREÇO (RUA/Nº):").pack(pady=2); ent_end = ttk.Entry(janela); ent_end.pack(fill="x", padx=20)
    tk.Label(janela, text="BAIRRO:").pack(pady=2); ent_bairro = ttk.Entry(janela); ent_bairro.pack(fill="x", padx=20)
    tk.Label(janela, text="COMPLEMENTO:").pack(pady=2); ent_compl = ttk.Entry(janela); ent_compl.pack(fill="x", padx=20)
    tk.Label(janela, text="TAXA ENTREGA (R$):").pack(pady=2); ent_taxa = ttk.Entry(janela); ent_taxa.insert(0, "0.00"); ent_taxa.pack(fill="x", padx=20)
    tk.Label(janela, text="TROCO PARA (R$):").pack(pady=2); ent_troco = ttk.Entry(janela); ent_troco.insert(0, "0.00"); ent_troco.pack(fill="x", padx=20)
    tk.Label(janela, text="OBSERVAÇÕES:").pack(pady=2); ent_obs = ttk.Entry(janela); ent_obs.pack(fill="x", padx=20)
    tk.Label(janela, text="MOTOBOY:").pack(pady=2); ent_moto = ttk.Entry(janela); ent_moto.pack(fill="x", padx=20)
    def confirmar():
        try:
            tx, tr = float(ent_taxa.get() or 0), float(ent_troco.get() or 0); dh = datetime.now().strftime("%d/%m/%Y %H:%M")
            vid = processar_venda_delivery_banco(itens, total + tx, "DELIVERY - " + pagamento)
            if vid:
                it_fmt = [(i['nome'], i['qtd'], i['preco'], i['sub']) for i in itens]
                f, _ = ImpressoraTermica.gerar_comanda_delivery_termica(vid, dh, ent_nome.get(), ent_tel.get(), ent_end.get(), ent_bairro.get(), ent_compl.get(), it_fmt, total, tx, pagamento, tr if tr > 0 else None, ent_obs.get(), ent_moto.get())
                ImpressoraTermica.imprimir_arquivo(f); messagebox.showinfo("Sucesso", "Delivery registrado!"); janela.destroy()
                for i in tree.get_children(): tree.delete(i)
                atualizar_total(); atualizar_badge_alerta()
        except: messagebox.showerror("Erro", "Valores inválidos")
    ttk.Button(janela, text="FINALIZAR E IMPRIMIR COMANDA", command=confirmar).pack(pady=20)

def processar_venda_delivery_banco(itens, total, pagamento):
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor(); data = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    cursor.execute("INSERT INTO vendas (data, total, pagamento) VALUES (?, ?, ?)", (data, total, pagamento)); vid = cursor.lastrowid
    for it in itens:
        cursor.execute("INSERT INTO itens_venda (venda_id, produto_id, nome_produto, quantidade, preco_unitario, subtotal, preco_custo_unitario) VALUES (?, ?, ?, ?, ?, ?, ?)", (vid, it['id'], it['nome'], it['qtd'], it['preco'], it['sub'], it['custo']))
        cursor.execute("UPDATE produtos SET estoque = estoque - ? WHERE id = ?", (it['qtd'], it['id']))
    conn.commit(); conn.close(); return vid

def finalizar_venda():
    if not tree.get_children():
        messagebox.showerror("Erro", "O carrinho está vazio."); return

    # Pergunta se deseja gerar comanda de Delivery
    if messagebox.askyesno("Delivery", "Deseja gerar comanda de Delivery para Motoboy?"):
        itens_venda = []
        total_venda = 0
        for item in tree.get_children():
            valores = tree.item(item)["values"]
            itens_venda.append({
                "id": valores[1], "nome": valores[2], "qtd": int(valores[3]),
                "preco": float(valores[4]), "sub": float(valores[5]), "custo": float(valores[6])
            })
            total_venda += float(valores[5])
        abrir_tela_delivery(itens_venda, total_venda, combo_pagamento.get())
    elif combo_pagamento.get() == "Fiado":
        buscar_cliente_fiado()
    else:
        executar_finalizacao_venda()

def executar_finalizacao_venda(cliente_id=None, cliente_nome=None):
    pagamento = combo_pagamento.get()
    total = sum(float(tree.item(i)["values"][5]) for i in tree.get_children())
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    try:
        data_atual = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        cursor.execute("INSERT INTO vendas (data, total, pagamento, cliente_id) VALUES (?, ?, ?, ?)",
                       (data_atual, total, pagamento, cliente_id))
        venda_id = cursor.lastrowid

        itens_para_recibo = []
        for item in tree.get_children():
            valores = tree.item(item)["values"]
            id_p, nome_p, qtd, preco, sub, custo = valores[1], valores[2], int(valores[3]), float(valores[4]), float(valores[5]), float(valores[6])
            cursor.execute("INSERT INTO itens_venda (venda_id, produto_id, nome_produto, quantidade, preco_unitario, subtotal, preco_custo_unitario) VALUES (?, ?, ?, ?, ?, ?, ?)",
                           (venda_id, id_p, nome_p, qtd, preco, sub, custo))
            cursor.execute("UPDATE produtos SET estoque = estoque - ? WHERE id = ?", (qtd, id_p))
            itens_para_recibo.append((nome_p, qtd, preco, sub))

        if pagamento == "Fiado" and cliente_id:
            cursor.execute("INSERT INTO debitos (cliente_id, venda_id, valor, status, data) VALUES (?, ?, ?, ?, ?)",
                           (cliente_id, venda_id, total, 'PENDENTE', data_atual))
        conn.commit()

        # Diálogo de Impressão
        janela_imp = tk.Toplevel(root); janela_imp.title("Impressão"); janela_imp.geometry("300x250")
        tk.Label(janela_imp, text="Venda Finalizada com Sucesso!", font=("Segoe UI", 12, "bold")).pack(pady=10)

        def imp_pdf():
            f = GeradorPDF.gerar_recibo_venda(venda_id, data_atual, total, pagamento, itens_para_recibo, cliente_nome)
            janela_imp.destroy(); messagebox.showinfo("Sucesso", f"Recibo PDF gerado: {f}")

        def enviar_direto_epson():
            f, conteudo = ImpressoraTermica.gerar_texto_venda(venda_id, data_atual, total, pagamento, itens_para_recibo, cliente_nome)
            if ImpressoraTermica.imprimir_arquivo(f):
                janela_imp.destroy(); messagebox.showinfo("Impressão", "Recibo enviado para a Epson TM-T20!")

        tk.Button(janela_imp, text="🖨️ Imprimir na Epson TM-T20", command=enviar_direto_epson, bg="#1E88E5", fg="white", font=("Segoe UI", 10, "bold"), pady=8).pack(pady=5, fill="x", padx=20)
        tk.Button(janela_imp, text="📄 Gerar Recibo PDF", command=imp_pdf, bg="#4CAF50", fg="white", font=("Segoe UI", 10)).pack(pady=5, fill="x", padx=20)
        tk.Button(janela_imp, text="Sair sem Imprimir", command=janela_imp.destroy, bg="#757575", fg="white").pack(pady=10, fill="x", padx=20)

        for item in tree.get_children(): tree.delete(item)
        atualizar_total(); atualizar_badge_alerta()
        root.after(300, lambda: mostrar_alerta_estoque(silencioso=True))
    except Exception as e:
        conn.rollback(); messagebox.showerror("Erro", f"Erro ao finalizar venda: {e}")
    finally:
        conn.close()

# =============================
# TELAS E DASHBOARD (ADMIN)
# =============================

def tela_gerenciar_usuarios():
    if CARGO_LOGADO != "admin":
        messagebox.showerror("Acesso Negado", "Apenas administradores podem acessar esta tela.")
        return
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    ttk.Label(main_frame, text="Gerenciamento de Usuários", font=("Segoe UI", 18, "bold")).pack(pady=10)
    tree_u = ttk.Treeview(main_frame, columns=("ID", "Usuário", "Cargo"), show="headings")
    for col in ["ID", "Usuário", "Cargo"]: tree_u.heading(col, text=col)
    tree_u.pack(pady=10, padx=10, fill="both", expand=True)
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT id, username, role FROM usuarios"); rows = cursor.fetchall(); conn.close()
    for r in rows: tree_u.insert("", "end", values=r)
    def remover_u():
        sel = tree_u.selection()
        if not sel: messagebox.showerror("Erro", "Selecione um usuário"); return
        u_id, u_nome = tree_u.item(sel[0])["values"][0], tree_u.item(sel[0])["values"][1]
        if u_nome == USUARIO_LOGADO: messagebox.showerror("Erro", "Você não pode se remover"); return
        if messagebox.askyesno("Confirmação", f"Remover usuário {u_nome}?"):
            conn = sqlite3.connect("sistema.db"); cursor = conn.cursor(); cursor.execute("DELETE FROM usuarios WHERE id=?", (u_id,)); conn.commit(); conn.close(); tela_gerenciar_usuarios()
    ttk.Button(main_frame, text="Remover Usuário Selecionado", command=remover_u).pack(pady=10)

def tela_faturamento():
    if CARGO_LOGADO != "admin":
        messagebox.showerror("Acesso Negado", "Apenas administradores podem acessar esta tela.")
        return
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    metricas = obter_metricas_faturamento()
    ttk.Label(main_frame, text="📊 Faturamento e Lucro", font=("Segoe UI", 18, "bold")).pack(pady=(12, 4))
    ttk.Label(main_frame, text=f"Atualizado em {datetime.now().strftime('%d/%m/%Y  %H:%M')}", font=("Segoe UI", 9), foreground="gray").pack()

    frame_cards = tk.Frame(main_frame, bg="#F0F0F0"); frame_cards.pack(fill="x", padx=10, pady=10)

    def _criar_card_faturamento(parent, titulo, emoji, valor, lucro, margem, n_vendas, ticket, variacao, label_ant, bg_cor, fg_cor):
        card = tk.Frame(parent, bg=bg_cor, padx=14, pady=12, relief="flat", bd=0)
        card.pack(side="left", expand=True, fill="both", padx=8, pady=4)
        tk.Label(card, text=f"{emoji}  {titulo}", bg=bg_cor, fg="white", font=("Segoe UI", 11, "bold")).pack(anchor="w")
        tk.Label(card, text=f"Venda: R$ {valor:,.2f}".replace(",", "X").replace(".", ",").replace("X", "."), bg=bg_cor, fg="white", font=("Segoe UI", 16, "bold")).pack(anchor="w", pady=(4, 0))
        tk.Label(card, text=f"Lucro: R$ {lucro:,.2f}".replace(",", "X").replace(".", ",").replace("X", "."), bg=bg_cor, fg="#4ADE80", font=("Segoe UI", 14, "bold")).pack(anchor="w")
        tk.Label(card, text=f"Margem: {margem:.1f}%", bg=bg_cor, fg="#FFD700", font=("Segoe UI", 10, "bold")).pack(anchor="w")
        frame_sub = tk.Frame(card, bg=bg_cor)
        frame_sub.pack(anchor="w", pady=(2, 4))
        tk.Label(frame_sub, text=f"🛒 {n_vendas} venda(s)", bg=bg_cor, fg="#DDDDDD", font=("Segoe UI", 9)).pack(side="left", padx=(0, 8))
        tk.Label(frame_sub, text=f"🎯 Ticket: R$ {ticket:.2f}", bg=bg_cor, fg="#DDDDDD", font=("Segoe UI", 9)).pack(side="left")
        if variacao is not None:
            seta  = "▲" if variacao >= 0 else "▼"
            cor_v = "#4ADE80" if variacao >= 0 else "#F87171"
            tk.Label(card, text=f"{seta} {abs(variacao):.1f}%  {label_ant}", bg=bg_cor, fg=cor_v, font=("Segoe UI", 9, "bold")).pack(anchor="w")
        else:
            tk.Label(card, text="— sem dados anteriores", bg=bg_cor, fg="#AAAAAA", font=("Segoe UI", 9)).pack(anchor="w")

    card_specs = [("Hoje", "📅", metricas["diario"], "#1565C0", "#1565C0"), ("Esta Semana", "📆", metricas["semanal"], "#2E7D32", "#2E7D32"), ("Este Mês", "🗓️", metricas["mensal"], "#6A1B9A", "#6A1B9A")]
    for titulo, emoji, m, bg, fg in card_specs: _criar_card_faturamento(frame_cards, titulo, emoji, m["fat"], m["lucro"], m["margem"], m["nv"], m["ticket"], m["var"], m["label_ant"], bg, fg)

    frame_grafico = tk.Frame(main_frame, bg="#F0F0F0"); frame_grafico.pack(fill="both", expand=True, padx=10, pady=4)
    aba_var = tk.StringVar(value="diario")

    def mostrar_grafico(tipo):
        for w in frame_grafico.winfo_children(): w.destroy()
        if tipo == "diario": labels, valores = obter_serie_diaria()
        elif tipo == "semanal": labels, valores = obter_serie_semanal()
        else: labels, valores = obter_serie_mensal()

        fig = plt.Figure(figsize=(11, 4), dpi=96, facecolor="#F0F0F0")
        ax = fig.add_subplot(111, facecolor="#F0F0F0")
        ax.bar(labels, valores, color="#4A90E2", alpha=0.8, width=0.6, edgecolor="#2171C1", linewidth=1)
        ax.set_title(f"Evolução de Faturamento ({tipo.capitalize()})", fontdict={"fontsize": 12, "fontweight": "bold", "color": "#333333"}, pad=15)
        ax.spines["top"].set_visible(False); ax.spines["right"].set_visible(False)
        ax.grid(axis="y", linestyle="--", alpha=0.3)
        canvas_plot = FigureCanvasTkAgg(fig, master=frame_grafico); canvas_plot.draw(); canvas_plot.get_widget().pack(fill="both", expand=True)

    frame_abas = tk.Frame(main_frame, bg="#F0F0F0"); frame_abas.pack(fill="x", padx=10)
    def criar_btn_aba(texto, tipo):
        def cmd():
            aba_var.set(tipo)
            for b in btns_aba: b.config(relief="sunken" if b["text"] == texto else "raised", bg="#4A90D9" if b["text"] == texto else "#DDDDDD", fg="white" if b["text"] == texto else "#333333")
            mostrar_grafico(tipo)
        return tk.Button(frame_abas, text=texto, command=cmd, font=("Segoe UI", 10, "bold"), bg="#DDDDDD", fg="#333333", relief="raised", padx=14, pady=4, bd=1, cursor="hand2")

    btn_d = criar_btn_aba("📅 Diário", "diario"); btn_s = criar_btn_aba("📆 Semanal", "semanal"); btn_m = criar_btn_aba("🗓️ Mensal", "mensal")

    def imprimir_relatorio_faturamento():
        f = f"relatorio_faturamento_{date.today()}.pdf"
        c = canvas.Canvas(f, pagesize=letter)
        width, height = letter
        c.setFont("Helvetica-Bold", 16)
        c.drawString(50, height - 50, "RELATÓRIO FINANCEIRO - ROTA DOS DRINKS")
        c.setFont("Helvetica", 12)
        c.drawString(50, height - 80, f"Resumo de Hoje: R$ {metricas['diario']['fat']:.2f}")
        c.drawString(50, height - 100, f"Lucro Hoje: R$ {metricas['diario']['lucro']:.2f}")
        c.drawString(50, height - 120, f"Vendas Hoje: {metricas['diario']['nv']}")
        c.line(50, height - 130, width - 50, height - 130)
        c.drawString(50, height - 150, f"Resumo Semanal: R$ {metricas['semanal']['fat']:.2f}")
        c.drawString(50, height - 170, f"Resumo Mensal: R$ {metricas['mensal']['fat']:.2f}")
        c.save()
        messagebox.showinfo("Sucesso", f"Relatório financeiro gerado: {f}")

    btn_print = tk.Button(frame_abas, text="🖨️ Imprimir Resumo", command=imprimir_relatorio_faturamento, font=("Segoe UI", 10, "bold"), bg="#4CAF50", fg="white", relief="raised", padx=14, pady=4, bd=1, cursor="hand2")
    btn_print.pack(side="right", padx=3)
    btns_aba = [btn_d, btn_s, btn_m]; btn_d.config(bg="#4A90D9", fg="white", relief="sunken"); btn_d.pack(side="left", padx=3); btn_s.pack(side="left", padx=3); btn_m.pack(side="left", padx=3); mostrar_grafico("diario")

def tela_relatorio_mais_vendidos():
    if CARGO_LOGADO != "admin":
        messagebox.showerror("Acesso Negado", "Apenas administradores podem acessar esta tela.")
        return
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT nome_produto, SUM(quantidade) as total_vendido FROM itens_venda GROUP BY nome_produto ORDER BY total_vendido DESC LIMIT 10")
    dados = cursor.fetchall(); conn.close()
    if not dados: ttk.Label(main_frame, text="Nenhuma venda registrada ainda.", font=("Segoe UI", 14)).pack(pady=50); return
    ttk.Label(main_frame, text="Top 10 Produtos Mais Vendidos", font=("Segoe UI", 18, "bold")).pack(pady=20)
    tree_rel = ttk.Treeview(main_frame, columns=("Produto", "Quantidade Total"), show="headings")
    for col in tree_rel["columns"]: tree_rel.heading(col, text=col)
    tree_rel.pack(pady=10, padx=20, fill="both", expand=True)
    for item in dados: tree_rel.insert("", "end", values=item)

def tela_historico():
    if CARGO_LOGADO != "admin":
        messagebox.showerror("Acesso Negado", "Apenas administradores podem acessar esta tela.")
        return
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    ttk.Label(main_frame, text="Histórico de Vendas", font=("Segoe UI", 18, "bold")).pack(pady=10)
    tree_hist = ttk.Treeview(main_frame, columns=("ID", "Data", "Total", "Pagamento"), show="headings")
    for col in tree_hist["columns"]: tree_hist.heading(col, text=col)
    tree_hist.pack(pady=10, padx=10, fill="both", expand=True)
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT id, data, total, pagamento FROM vendas ORDER BY id DESC")
    for row in cursor.fetchall(): tree_hist.insert("", "end", values=row)
    conn.close()

def tela_debitos():
    """Tela de controle de débitos fiados com todos os botões funcionando"""
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    ttk.Label(main_frame, text="Controle de Fiados (Débitos)", font=("Segoe UI", 18, "bold")).pack(pady=10)
    global tree_deb
    tree_deb = ttk.Treeview(main_frame, columns=("Sel", "ID", "Cliente", "Valor", "Data", "Status"), show="headings")
    tree_deb.heading("Sel", text="[ ]")
    tree_deb.column("Sel", width=40, anchor="center")
    for col in ["ID", "Cliente", "Valor", "Data", "Status"]: tree_deb.heading(col, text=col)
    tree_deb.pack(pady=10, padx=10, fill="both", expand=True)
    tree_deb.bind("<Button-1>", lambda e: toggle_checkbox(e, tree_deb))

    def carregar_debitos():
        """Carrega os débitos pendentes na tabela"""
        tree_deb.delete(*tree_deb.get_children())
        conn = sqlite3.connect("sistema.db")
        cursor = conn.cursor()

        try:
            cursor.execute("""
                SELECT d.id, c.nome, d.valor, d.data, d.status 
                FROM debitos d 
                JOIN clientes c ON d.cliente_id = c.id 
                WHERE d.status = 'PENDENTE' 
                ORDER BY d.id DESC
            """)
            for row in cursor.fetchall():
                tree_deb.insert("", "end", values=("☐", *row))
        except Exception as e:
            print(f"[DEBUG] Erro ao carregar débitos: {e}")
            try:
                cursor.execute("""
                    SELECT d.id, c.nome, d.valor, datetime('now'), d.status 
                    FROM debitos d 
                    JOIN clientes c ON d.cliente_id = c.id 
                    WHERE d.status = 'PENDENTE' 
                    ORDER BY d.id DESC
                """)
                for row in cursor.fetchall():
                    tree_deb.insert("", "end", values=("☐", *row))
            except Exception as e2:
                messagebox.showerror("Erro", f"Erro ao carregar débitos: {e2}")
        finally:
            conn.close()

    def quitar_debitos_selecionados():
        """Marca débitos selecionados como PAGO"""
        selecionados = [tree_deb.item(i)["values"][1] for i in tree_deb.get_children() if tree_deb.item(i)["values"][0] == "☑"]
        if not selecionados:
            messagebox.showerror("Erro", "Selecione ao menos um débito")
            return
        if messagebox.askyesno("Confirmação", f"Confirmar pagamento de {len(selecionados)} débito(s)?"):
            conn = sqlite3.connect("sistema.db")
            cursor = conn.cursor()
            for id_d in selecionados:
                cursor.execute("UPDATE debitos SET status = 'PAGO' WHERE id = ?", (id_d,))
            conn.commit()
            conn.close()
            carregar_debitos()
            messagebox.showinfo("Sucesso", "Débitos quitados!")

    def quitar_debito_parcial():
        """Função para quitar um débito com valor parcial personalizável e atualização visual"""
        # Verificar se há débitos selecionados
        # Corrigido: Pegar o item selecionado (focado) se nenhum estiver marcado com checkbox, 
        # ou pegar o marcado com checkbox.
        selecionados_itens = [i for i in tree_deb.get_children() if tree_deb.item(i)["values"][0] == "☑"]
        
        if not selecionados_itens:
            focado = tree_deb.focus()
            if focado:
                selecionados_itens = [focado]
            else:
                messagebox.showerror("Erro", "Selecione ao menos um débito para quitar parcialmente")
                return

        if len(selecionados_itens) > 1:
            messagebox.showerror("Erro", "Selecione apenas um débito por vez para pagamento parcial")
            return

        # Extrair informações do débito selecionado
        item_id = selecionados_itens[0]
        debito_info = tree_deb.item(item_id)["values"]
        id_debito = debito_info[1]  # ID do débito
        cliente_nome = debito_info[2]  # Nome do cliente
        
        # Garantir que o valor total seja lido corretamente do banco para evitar discrepâncias visuais
        conn = sqlite3.connect("sistema.db")
        cursor = conn.cursor()
        cursor.execute("SELECT valor FROM debitos WHERE id = ?", (id_debito,))
        res = cursor.fetchone()
        conn.close()
        
        if res:
            valor_total = float(res[0])
        else:
            valor_total = float(debito_info[3])

        # Criar janela de diálogo para pagamento parcial
        janela_pagto = tk.Toplevel(root)
        janela_pagto.title("Pagamento Parcial de Débito")
        janela_pagto.geometry("500x480")
        janela_pagto.resizable(False, False)

        # Centralizar janela
        janela_pagto.transient(root)
        janela_pagto.grab_set()

        # Título
        tk.Label(janela_pagto, text="Pagamento Parcial de Débito", font=("Segoe UI", 14, "bold"), fg="#1E88E5").pack(pady=15)

        # Informações do cliente
        frame_info = tk.Frame(janela_pagto, bg="#F5F5F5", relief="flat", bd=1)
        frame_info.pack(pady=10, padx=15, fill="x")

        tk.Label(frame_info, text=f"Cliente: {cliente_nome}", font=("Segoe UI", 11, "bold"), bg="#F5F5F5").pack(anchor="w", padx=10, pady=5)
        tk.Label(frame_info, text=f"Débito Total: R$ {valor_total:.2f}", font=("Segoe UI", 11, "bold"), bg="#F5F5F5", fg="#D32F2F").pack(anchor="w", padx=10, pady=5)

        # Separador
        ttk.Separator(janela_pagto, orient="horizontal").pack(fill="x", pady=5)

        # Campo de entrada do valor a pagar
        tk.Label(janela_pagto, text="Valor a Pagar:", font=("Segoe UI", 11, "bold")).pack(anchor="w", padx=15, pady=(10, 5))

        frame_entrada = tk.Frame(janela_pagto)
        frame_entrada.pack(padx=15, fill="x", pady=5)

        tk.Label(frame_entrada, text="R$", font=("Segoe UI", 11, "bold")).pack(side="left", padx=(0, 5))

        entry_valor_pagto = ttk.Entry(frame_entrada, font=("Segoe UI", 11), width=20)
        entry_valor_pagto.pack(side="left", fill="x", expand=True)
        entry_valor_pagto.focus()

        def confirmar_valor():
            """Confirma o valor digitado, atualiza o banco de dados e a tela principal"""
            try:
                valor_pagto = float(entry_valor_pagto.get().replace(",", "."))

                # Validações
                if valor_pagto <= 0:
                    messagebox.showerror("Erro", "Digite um valor maior que zero!")
                    entry_valor_pagto.delete(0, tk.END)
                    entry_valor_pagto.focus()
                    return

                if valor_pagto > valor_total:
                    messagebox.showerror("Erro", f"O valor não pode ser maior que o débito total (R$ {valor_total:.2f})")
                    entry_valor_pagto.delete(0, tk.END)
                    entry_valor_pagto.focus()
                    return

                if messagebox.askyesno("Confirmação", f"Confirmar o pagamento parcial de R$ {valor_pagto:.2f}?\nO saldo será atualizado imediatamente."):
                    conn = sqlite3.connect("sistema.db")
                    cursor = conn.cursor()

                    if valor_pagto == valor_total:
                        cursor.execute("UPDATE debitos SET status = 'PAGO', valor = 0 WHERE id = ?", (id_debito,))
                    else:
                        cursor.execute("UPDATE debitos SET valor = valor - ? WHERE id = ?", (valor_pagto, id_debito))

                    conn.commit()
                    conn.close()

                    # Atualizar a tela principal e fechar esta janela
                    carregar_debitos()
                    janela_pagto.destroy()
                    messagebox.showinfo("Sucesso", f"Pagamento de R$ {valor_pagto:.2f} registrado com sucesso!")

            except ValueError:
                messagebox.showerror("Erro", "Digite um valor válido!\nUse ponto (.) ou vírgula (,) como separador decimal")
                entry_valor_pagto.delete(0, tk.END)
                entry_valor_pagto.focus()

        # Frame para entrada + botão confirmar
        frame_botao_confirmar = tk.Frame(janela_pagto)
        frame_botao_confirmar.pack(padx=15, fill="x", pady=5)

        tk.Button(frame_botao_confirmar, text="✓ Confirmar Pagamento", command=confirmar_valor, 
                  bg="#4CAF50", fg="white", font=("Segoe UI", 11, "bold"), 
                  relief="raised", bd=1, cursor="hand2").pack(side="left", padx=(0, 5), fill="x", expand=True)

        # Botões de atalho para valores comuns
        frame_atalhos = tk.Frame(janela_pagto)
        frame_atalhos.pack(padx=15, pady=10, fill="x")

        tk.Label(frame_atalhos, text="Atalhos Rápidos:", font=("Segoe UI", 9, "bold")).pack(anchor="w", pady=(0, 5))

        frame_botoes_atalho = tk.Frame(janela_pagto)
        frame_botoes_atalho.pack(padx=15, fill="x", pady=5)

        def preencher_valor(valor):
            entry_valor_pagto.delete(0, tk.END)
            entry_valor_pagto.insert(0, str(valor))
            atualizar_resumo()

        def valor_25():
            preencher_valor(f"{valor_total * 0.25:.2f}")

        def valor_50():
            preencher_valor(f"{valor_total * 0.5:.2f}")

        def valor_75():
            preencher_valor(f"{valor_total * 0.75:.2f}")

        def valor_total_btn():
            preencher_valor(f"{valor_total:.2f}")

        tk.Button(frame_botoes_atalho, text="25%", command=valor_25, bg="#E0E0E0", font=("Segoe UI", 9), width=8).pack(side="left", padx=2)
        tk.Button(frame_botoes_atalho, text="50%", command=valor_50, bg="#E0E0E0", font=("Segoe UI", 9), width=8).pack(side="left", padx=2)
        tk.Button(frame_botoes_atalho, text="75%", command=valor_75, bg="#E0E0E0", font=("Segoe UI", 9), width=8).pack(side="left", padx=2)
        tk.Button(frame_botoes_atalho, text="100%", command=valor_total_btn, bg="#4CAF50", fg="white", font=("Segoe UI", 9, "bold"), width=8).pack(side="left", padx=2)

        # Separador
        ttk.Separator(janela_pagto, orient="horizontal").pack(fill="x", pady=10)

        # Resumo do pagamento
        frame_resumo = tk.Frame(janela_pagto, bg="#F5F5F5", relief="flat", bd=1)
        frame_resumo.pack(pady=10, padx=15, fill="x")

        lbl_novo_saldo = tk.Label(frame_resumo, text=f"Novo Saldo Devedor: R$ {valor_total:.2f}", font=("Segoe UI", 11, "bold"), bg="#F5F5F5", fg="#2E7D32")
        lbl_novo_saldo.pack(pady=10)

        def atualizar_resumo(event=None):
            try:
                texto = entry_valor_pagto.get().replace(",", ".").strip()
                if not texto:
                    lbl_novo_saldo.config(text=f"Novo Saldo Devedor: R$ {valor_total:.2f}", fg="#2E7D32")
                    return
                
                v = float(texto)
                novo = valor_total - v
                
                if novo < 0:
                    lbl_novo_saldo.config(text="Valor maior que o débito!", fg="#D32F2F")
                else:
                    lbl_novo_saldo.config(text=f"Novo Saldo Devedor: R$ {novo:.2f}", fg="#2E7D32")
            except ValueError:
                lbl_novo_saldo.config(text="Digite um valor válido", fg="#D32F2F")

        # Vincular atualização do resumo ao digitar
        entry_valor_pagto.bind("<KeyRelease>", atualizar_resumo)

        # Botão Fechar
        frame_acoes = tk.Frame(janela_pagto)
        frame_acoes.pack(pady=20, padx=15, fill="x")

        tk.Button(frame_acoes, text="FECHAR / CANCELAR", command=janela_pagto.destroy, 
                  bg="#F44336", fg="white", font=("Segoe UI", 11, "bold"), 
                  relief="raised", bd=2, cursor="hand2", pady=8).pack(fill="x", expand=True)
    def imprimir_debitos_termica():
        """Imprime débitos na impressora térmica Epson"""
        selecionados = [tree_deb.item(i)["values"][1:5] for i in tree_deb.get_children() if tree_deb.item(i)["values"][0] == "☑"]
        if not selecionados:
            messagebox.showerror("Erro", "Selecione ao menos um débito para imprimir")
            return
        f, conteudo = ImpressoraTermica.gerar_texto_debitos(selecionados)
        if ImpressoraTermica.imprimir_arquivo(f):
            messagebox.showinfo("Sucesso", "Relatório de débitos enviado para a Epson TM-T20!")

    # =============================
    # CRIAR BOTÕES
    # =============================
    frame_botoes = ttk.Frame(main_frame)
    frame_botoes.pack(pady=10)

    ttk.Button(frame_botoes, text="Quitar Selecionados", command=quitar_debitos_selecionados).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="Quitar Parcialmente", command=quitar_debito_parcial).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="🖨️ Imprimir Débitos (Epson)", command=imprimir_debitos_termica).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="Selecionar Todos", command=lambda: select_all(tree_deb)).pack(side="left", padx=5)

    # =============================
    # CARREGAR DADOS
    # =============================
    carregar_debitos()


def atualizar_dashboard():
    if not dashboard_ativo: return
    if CARGO_LOGADO != 'admin':
        tela_venda()
        return
    for widget in main_frame.winfo_children(): widget.destroy()
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT SUM(total) FROM vendas"); total_geral = cursor.fetchone()[0] or 0
    cursor.execute("SELECT SUM(quantidade * preco_custo_unitario) FROM itens_venda"); custo_total = cursor.fetchone()[0] or 0
    lucro_total = total_geral - custo_total
    cursor.execute("SELECT data FROM vendas ORDER BY id DESC LIMIT 1"); row = cursor.fetchone(); ultima_venda = row[0] if row else "Nenhuma"
    cursor.execute("SELECT COUNT(*) FROM produtos WHERE estoque <= ?", (LIMITE_ESTOQUE_BAIXO,)); qtd_alertas = cursor.fetchone()[0]
    cursor.execute("SELECT COUNT(*) FROM produtos WHERE estoque = 0"); qtd_zerados = cursor.fetchone()[0]
    cursor.execute("SELECT substr(data,1,10), SUM(total) FROM vendas GROUP BY substr(data,1,10) ORDER BY substr(data,1,10)"); dados_grafico = cursor.fetchall(); conn.close()
    metricas = obter_metricas_faturamento()
    frame_topo = tk.Frame(main_frame, bg="#1E1E2E", pady=8); frame_topo.pack(fill="x")
    tk.Label(frame_topo, text=f"💰 Faturamento Total: R$ {total_geral:,.2f}".replace(",","X").replace(".",",").replace("X","."), bg="#1E1E2E", fg="#4CAF50", font=("Segoe UI", 17, "bold")).pack()
    tk.Label(frame_topo, text=f"📈 Lucro Total: R$ {lucro_total:,.2f}".replace(",","X").replace(".",",").replace("X","."), bg="#1E1E2E", fg="#FFD700", font=("Segoe UI", 14, "bold")).pack()
    tk.Label(frame_topo, text=f"🕒 Última Venda: {ultima_venda}", bg="#1E1E2E", fg="gray", font=("Segoe UI", 11)).pack()

    def _criar_card_faturamento(parent, titulo, emoji, valor, lucro, margem, n_vendas, ticket, variacao, label_ant, bg_cor, fg_cor):
        card = tk.Frame(parent, bg=bg_cor, padx=14, pady=12, relief="flat", bd=0)
        card.pack(side="left", expand=True, fill="both", padx=8, pady=4)
        tk.Label(card, text=f"{emoji}  {titulo}", bg=bg_cor, fg="white", font=("Segoe UI", 11, "bold")).pack(anchor="w")
        tk.Label(card, text=f"Venda: R$ {valor:,.2f}".replace(",", "X").replace(".", ",").replace("X", "."), bg=bg_cor, fg="white", font=("Segoe UI", 16, "bold")).pack(anchor="w", pady=(4, 0))
        tk.Label(card, text=f"Lucro: R$ {lucro:,.2f}".replace(",", "X").replace(".", ",").replace("X", "."), bg=bg_cor, fg="#4ADE80", font=("Segoe UI", 14, "bold")).pack(anchor="w")
        tk.Label(card, text=f"Margem: {margem:.1f}%", bg=bg_cor, fg="#FFD700", font=("Segoe UI", 10, "bold")).pack(anchor="w")
        frame_sub = tk.Frame(card, bg=bg_cor)
        frame_sub.pack(anchor="w", pady=(2, 4))
        tk.Label(frame_sub, text=f"🛒 {n_vendas} venda(s)", bg=bg_cor, fg="#DDDDDD", font=("Segoe UI", 9)).pack(side="left", padx=(0, 8))
        tk.Label(frame_sub, text=f"🎯 Ticket: R$ {ticket:.2f}", bg=bg_cor, fg="#DDDDDD", font=("Segoe UI", 9)).pack(side="left")
        if variacao is not None:
            seta  = "▲" if variacao >= 0 else "▼"
            cor_v = "#4ADE80" if variacao >= 0 else "#F87171"
            tk.Label(card, text=f"{seta} {abs(variacao):.1f}%  {label_ant}", bg=bg_cor, fg=cor_v, font=("Segoe UI", 9, "bold")).pack(anchor="w")
        else:
            tk.Label(card, text="— sem dados anteriores", bg=bg_cor, fg="#AAAAAA", font=("Segoe UI", 9)).pack(anchor="w")

    frame_cards = tk.Frame(main_frame, bg="#2C2C3E", pady=10, padx=6); frame_cards.pack(fill="x")
    card_specs = [("Hoje", "📅", metricas["diario"], "#1565C0"), ("Esta Semana", "📆", metricas["semanal"], "#2E7D32"), ("Este Mês", "🗓️", metricas["mensal"], "#6A1B9A")]
    for titulo, emoji, m, bg in card_specs: _criar_card_faturamento(frame_cards, titulo, emoji, m["fat"], m["lucro"], m["margem"], m["nv"], m["ticket"], m["var"], m["label_ant"], bg, bg)
    cor_card = "#B71C1C" if qtd_zerados > 0 else ("#E65100" if qtd_alertas > 0 else "#1B5E20")
    frame_alerta_dash = tk.Frame(main_frame, bg=cor_card, pady=6, padx=15); frame_alerta_dash.pack(fill="x", padx=12, pady=(4, 0))
    texto_alerta = ("✅  Todos os produtos com estoque adequado" if qtd_alertas == 0 else f"⚠️  {qtd_alertas} produto(s) com estoque baixo   |   💀 {qtd_zerados} produto(s) zerado(s)")
    lbl_dash = tk.Label(frame_alerta_dash, text=texto_alerta, bg=cor_card, fg="white", font=("Segoe UI", 10, "bold"), cursor="hand2"); lbl_dash.pack(side="left")
    if qtd_alertas > 0:
        tk.Label(frame_alerta_dash, text="  [clique para ver]", bg=cor_card, fg="#FFE0B2", font=("Segoe UI", 9), cursor="hand2").pack(side="left")
        frame_alerta_dash.bind("<Button-1>", lambda e: mostrar_alerta_estoque()); lbl_dash.bind("<Button-1>", lambda e: mostrar_alerta_estoque())
    if dados_grafico:
        dias = [d[0] for d in dados_grafico[-30:]]; totais = [d[1] for d in dados_grafico[-30:]]
        fig, ax = plt.subplots(figsize=(11, 3.5), dpi=95); fig.patch.set_facecolor("#F0F0F0"); ax.set_facecolor("#FAFAFA"); ax.bar(dias, totais, color="#1E88E5", alpha=0.7, zorder=3); ax.plot(dias, totais, marker="o", color="#FF6B35", linewidth=1.5, zorder=4); ax.set_title("Faturamento por Dia (últimos registros)", fontsize=11, fontweight="bold"); ax.set_ylabel("R$"); ax.yaxis.set_major_formatter(plt.FuncFormatter(lambda x, _: f"R${x:,.0f}".replace(",","."))); plt.xticks(rotation=40, ha="right", fontsize=7); ax.grid(axis="y", linestyle="--", alpha=0.5); fig.tight_layout(); canvas = FigureCanvasTkAgg(fig, main_frame); canvas.draw(); canvas.get_tk_widget().pack(fill="both", expand=True, padx=12, pady=6); plt.close(fig)
    else: ttk.Label(main_frame, text="Nenhuma venda registrada.", font=("Segoe UI", 16)).pack(pady=40)

def tela_dashboard():
    if CARGO_LOGADO != 'admin':
        tela_venda()
        return
    global dashboard_ativo; dashboard_ativo = True; limpar_tela(); atualizar_dashboard()

def carregar_estoque():
    tree_est.delete(*tree_est.get_children())
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    cursor.execute("SELECT id, nome, preco, estoque, preco_custo FROM produtos")
    for item in cursor.fetchall():
        tag = "zerado" if item[3] == 0 else ("critico" if item[3] <= 2 else ("baixo" if item[3] <= LIMITE_ESTOQUE_BAIXO else ""))
        tree_est.insert("", "end", values=("☐", item[0], item[1], item[2], item[3], item[4]), tags=(tag,))
    conn.close(); atualizar_badge_alerta()

def alterar_estoque_selecionados():
    selecionados = [tree_est.item(i)["values"][1] for i in tree_est.get_children() if tree_est.item(i)["values"][0] == "☑"]
    if not selecionados: messagebox.showerror("Erro", "Selecione ao menos um produto"); return
    janela = tk.Toplevel(root); janela.title("Alterar Estoque"); janela.geometry("300x150")
    tk.Label(janela, text="Quantidade a Adicionar:").pack(pady=5)
    entry_qtd = ttk.Entry(janela); entry_qtd.pack(pady=5)
    def salvar():
        try:
            qtd = int(entry_qtd.get()); conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
            for id_p in selecionados: cursor.execute("UPDATE produtos SET estoque = estoque + ? WHERE id = ?", (qtd, id_p))
            conn.commit(); conn.close(); carregar_estoque(); janela.destroy(); messagebox.showinfo("Sucesso", "Estoque atualizado!"); atualizar_badge_alerta()
        except: messagebox.showerror("Erro", "Quantidade inválida")
    ttk.Button(janela, text="Salvar", command=salvar).pack(pady=5)

def remover_produtos_selecionados():
    selecionados = [tree_est.item(i)["values"][1] for i in tree_est.get_children() if tree_est.item(i)["values"][0] == "☑"]
    if not selecionados: messagebox.showerror("Erro", "Selecione ao menos um produto"); return
    if messagebox.askyesno("Confirmação", f"Deseja realmente remover {len(selecionados)} produto(s)?"):
        conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
        for id_p in selecionados: cursor.execute("DELETE FROM produtos WHERE id = ?", (id_p,))
        conn.commit(); conn.close(); carregar_estoque(); messagebox.showinfo("Sucesso", "Produtos removidos!"); atualizar_badge_alerta()

def limpar_tela():
    for widget in main_frame.winfo_children(): widget.destroy()

def tela_cadastro():
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    
    # Criar um PanedWindow para dividir a tela se necessário, ou apenas frames
    main_container = ttk.Frame(main_frame)
    main_container.pack(fill="both", expand=True)
    
    # Aba Lateral de Geração de Código (Sidebar)
    sidebar_width = 250
    sidebar = tk.Frame(main_container, bg="#E0E0E0", width=sidebar_width)
    sidebar.pack(side="right", fill="y", padx=(0, 10), pady=10)
    sidebar.pack_propagate(False)
    
    tk.Label(sidebar, text="🛠️ Ferramentas", font=("Segoe UI", 12, "bold"), bg="#E0E0E0").pack(pady=10)
    tk.Label(sidebar, text="Produtos sem código de barras:", font=("Segoe UI", 9), bg="#E0E0E0", wraplength=200).pack(pady=5, padx=10)
    
    btn_gerar_cod = tk.Button(sidebar, text="🔢 GERAR CÓDIGO INTERNO", command=gerar_codigo_interno, 
                             bg="#2196F3", fg="white", font=("Segoe UI", 9, "bold"), pady=10, cursor="hand2")
    btn_gerar_cod.pack(pady=10, padx=10, fill="x")
    
    tk.Label(sidebar, text="O código gerado será inserido automaticamente no campo 'Código de Barras'.", 
             font=("Segoe UI", 8, "italic"), bg="#E0E0E0", fg="#666666", wraplength=200).pack(pady=5, padx=10)

    # Conteúdo Principal (Notebook)
    nb = ttk.Notebook(main_container); nb.pack(side="left", pady=10, padx=10, fill="both", expand=True)
    
    f_prod_aba = ttk.Frame(nb); nb.add(f_prod_aba, text="Produtos")
    frame_prod = ttk.LabelFrame(f_prod_aba, text="Cadastrar Novo Produto"); frame_prod.pack(pady=10, padx=10, fill="x")
    
    ttk.Label(frame_prod, text="Código de Barras:").grid(row=0, column=0, padx=5, pady=5, sticky="w")
    global entry_cod_barras; entry_cod_barras = ttk.Entry(frame_prod); entry_cod_barras.grid(row=0, column=1, padx=5, pady=5, sticky="ew")
    
    ttk.Label(frame_prod, text="Nome do Produto:").grid(row=1, column=0, padx=5, pady=5, sticky="w")
    global entry_nome_prod; entry_nome_prod = ttk.Entry(frame_prod); entry_nome_prod.grid(row=1, column=1, padx=5, pady=5, sticky="ew")
    
    ttk.Label(frame_prod, text="Preço de Venda:").grid(row=2, column=0, padx=5, pady=5, sticky="w")
    global entry_preco; entry_preco = ttk.Entry(frame_prod); entry_preco.grid(row=2, column=1, padx=5, pady=5, sticky="ew")
    
    ttk.Label(frame_prod, text="Preço de Custo:").grid(row=3, column=0, padx=5, pady=5, sticky="w")
    global entry_custo; entry_custo = ttk.Entry(frame_prod); entry_custo.grid(row=3, column=1, padx=5, pady=5, sticky="ew")
    
    ttk.Label(frame_prod, text="Estoque Inicial:").grid(row=4, column=0, padx=5, pady=5, sticky="w")
    global entry_estoque; entry_estoque = ttk.Entry(frame_prod); entry_estoque.grid(row=4, column=1, padx=5, pady=5, sticky="ew")
    
    ttk.Button(frame_prod, text="Cadastrar Produto", command=cadastrar_produto).grid(row=5, column=0, columnspan=2, pady=10)
    
    f_cli_aba = ttk.Frame(nb); nb.add(f_cli_aba, text="Clientes")
    frame_cli = ttk.LabelFrame(f_cli_aba, text="Cadastrar Novo Cliente (Fiado)"); frame_cli.pack(pady=10, padx=10, fill="x")
    ttk.Label(frame_cli, text="Nome do Cliente:").grid(row=0, column=0, padx=5, pady=5, sticky="w")
    global entry_nome_cli; entry_nome_cli = ttk.Entry(frame_cli); entry_nome_cli.grid(row=0, column=1, padx=5, pady=5, sticky="ew")
    ttk.Label(frame_cli, text="Telefone:").grid(row=1, column=0, padx=5, pady=5, sticky="w")
    global entry_tel_cli; entry_tel_cli = ttk.Entry(frame_cli); entry_tel_cli.grid(row=1, column=1, padx=5, pady=5, sticky="ew")
    ttk.Label(frame_cli, text="Limite Fiado:").grid(row=2, column=0, padx=5, pady=5, sticky="w")
    global entry_limite_cli; entry_limite_cli = ttk.Entry(frame_cli); entry_limite_cli.grid(row=2, column=1, padx=5, pady=5, sticky="ew")
    ttk.Button(frame_cli, text="Cadastrar Cliente", command=cadastrar_cliente).grid(row=3, column=0, columnspan=2, pady=10)
    
    ttk.Label(f_cli_aba, text="Gerenciar Clientes Cadastrados", font=("Segoe UI", 12, "bold")).pack(pady=(10,0))
    global tree_cli_list
    tree_cli_list = ttk.Treeview(f_cli_aba, columns=("ID", "Nome", "Telefone", "Limite"), show="headings")
    for col in ["ID", "Nome", "Telefone", "Limite"]: tree_cli_list.heading(col, text=col)
    tree_cli_list.pack(pady=10, padx=10, fill="both", expand=True)
    
    f_btns_cli = ttk.Frame(f_cli_aba); f_btns_cli.pack(pady=5)
    ttk.Button(f_btns_cli, text="Editar Selecionado", command=editar_cliente_selecionado).pack(side="left", padx=5)
    ttk.Button(f_btns_cli, text="Remover Selecionado", command=remover_cliente_selecionado).pack(side="left", padx=5)
    carregar_clientes_lista()

def remover_itens_venda_selecionados():
    selecionados = [i for i in tree.get_children() if tree.item(i)["values"][0] == "☑"]
    if not selecionados: messagebox.showerror("Erro", "Selecione ao menos um item para remover"); return
    for item in selecionados: tree.delete(item)
    atualizar_total()

def toggle_checkbox(event, tree_widget):
    region = tree_widget.identify_region(event.x, event.y)
    if region == "cell" and tree_widget.identify_column(event.x) == "#1":
        item = tree_widget.identify_row(event.y)
        if item:
            values = list(tree_widget.item(item)["values"])
            values[0] = "☑" if values[0] == "☐" else "☐"
            tree_widget.item(item, values=values)

def select_all(tree_widget):
    for item in tree_widget.get_children():
        values = list(tree_widget.item(item)["values"])
        values[0] = "☑"
        tree_widget.item(item, values=values)

# --- FUNÇÕES PARA BUSCA INTELIGENTE ---
def on_key_release_codigo(event):
    if event.keysym in ["Up", "Down", "Return", "Escape"]: return

    termo = entry_codigo.get().strip()
    if len(termo) < 2:
        if hasattr(root, 'listbox_busca'): root.listbox_busca.place_forget()
        return

    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
    # Busca apenas produtos com estoque > 0
    cursor.execute("SELECT nome FROM produtos WHERE (nome LIKE ? OR codigo_barras LIKE ?) AND estoque > 0 LIMIT 8", (f"%{termo}%", f"%{termo}%"))
    sugestoes = [r[0] for r in cursor.fetchall()]; conn.close()

    if sugestoes:
        if not hasattr(root, 'listbox_busca'):
            root.listbox_busca = tk.Listbox(main_frame, font=("Segoe UI", 11), height=len(sugestoes))
            root.listbox_busca.bind("<<ListboxSelect>>", on_select_sugestao)

        root.listbox_busca.delete(0, tk.END)
        for s in sugestoes: root.listbox_busca.insert(tk.END, s)

        # Posiciona o listbox logo abaixo do campo de código
        x = entry_codigo.winfo_x()
        y = entry_codigo.winfo_y() + entry_codigo.winfo_height()
        root.listbox_busca.place(x=x, y=y, width=entry_codigo.winfo_width())
        root.listbox_busca.lift()
    else:
        if hasattr(root, 'listbox_busca'): root.listbox_busca.place_forget()

def on_select_sugestao(event):
    if not root.listbox_busca.curselection(): return
    selecionado = root.listbox_busca.get(root.listbox_busca.curselection())
    entry_codigo.delete(0, tk.END)
    entry_codigo.insert(0, selecionado)
    root.listbox_busca.place_forget()
    entry_quantidade.focus_set()

def tela_venda():
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    frame_entrada = ttk.LabelFrame(main_frame, text="Adicionar Produto à Venda"); frame_entrada.pack(pady=10, padx=10, fill="x")

    ttk.Label(frame_entrada, text="Busca / Cód. Barras:").grid(row=0, column=0, padx=5, pady=5, sticky="w")
    global entry_codigo; entry_codigo = ttk.Entry(frame_entrada); entry_codigo.grid(row=0, column=1, padx=5, pady=5, sticky="ew")
    entry_codigo.bind("<KeyRelease>", on_key_release_codigo)

    ttk.Label(frame_entrada, text="Quantidade:").grid(row=1, column=0, padx=5, pady=5, sticky="w")
    global entry_quantidade; entry_quantidade = ttk.Entry(frame_entrada); entry_quantidade.grid(row=1, column=1, padx=5, pady=5, sticky="ew")
    entry_quantidade.insert(0, "1")

    ttk.Button(frame_entrada, text="Adicionar ao Carrinho", command=adicionar_produto_venda).grid(row=2, column=0, columnspan=2, pady=5)

    global tree
    tree = ttk.Treeview(main_frame, columns=("Sel", "ID", "Produto", "Quantidade", "Preço Unit.", "Subtotal", "Custo"), show="headings")
    tree.heading("Sel", text="[ ]"); tree.column("Sel", width=40, anchor="center")
    for col in ["ID", "Produto", "Quantidade", "Preço Unit.", "Subtotal"]: tree.heading(col, text=col)
    tree.heading("Custo", text="Custo"); tree.column("Custo", width=0, stretch=False)
    tree.pack(pady=10, padx=10, fill="both", expand=True); tree.bind("<Button-1>", lambda e: toggle_checkbox(e, tree))
    ttk.Button(main_frame, text="Remover Itens Selecionados", command=remover_itens_venda_selecionados).pack(pady=5)
    global label_total; label_total = ttk.Label(main_frame, text="Total: R$ 0.00", font=("Segoe UI", 16, "bold")); label_total.pack(pady=5)
    frame_pagamento = ttk.Frame(main_frame); frame_pagamento.pack(pady=5)
    ttk.Label(frame_pagamento, text="Forma de Pagamento:").pack(side="left", padx=5)
    global combo_pagamento; combo_pagamento = ttk.Combobox(frame_pagamento, values=["Dinheiro", "Cartão de Crédito", "Cartão de Débito", "Pix", "Fiado"]); combo_pagamento.set("Dinheiro"); combo_pagamento.pack(side="left", padx=5)
    ttk.Button(frame_pagamento, text="Finalizar Venda", command=finalizar_venda).pack(side="left", padx=5)

def tela_estoque():
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()
    produtos_alerta = verificar_produtos_estoque_baixo()
    if produtos_alerta:
        frame_banner = tk.Frame(main_frame, bg="#FF6B35", pady=6); frame_banner.pack(fill="x")
        tk.Label(frame_banner, text=f"⚠️ ATENÇÃO: {len(produtos_alerta)} produto(s) com estoque baixo ou zerado!", bg="#FF6B35", fg="white", font=("Segoe UI", 10, "bold")).pack()
    ttk.Label(main_frame, text="Controle de Estoque", font=("Segoe UI", 18, "bold")).pack(pady=10)
    global tree_est
    tree_est = ttk.Treeview(main_frame, columns=("Sel", "ID", "Produto", "Preço Venda", "Estoque", "Preço Custo"), show="headings")
    tree_est.heading("Sel", text="[ ]"); tree_est.column("Sel", width=40, anchor="center")
    for col in ["ID", "Produto", "Preço Venda", "Estoque", "Preço Custo"]: tree_est.heading(col, text=col)
    tree_est.pack(pady=10, padx=10, fill="both", expand=True); tree_est.tag_configure("zerado", foreground="red", font=("Segoe UI", 9, "bold")); tree_est.tag_configure("critico", foreground="#D32F2F"); tree_est.tag_configure("baixo", foreground="#EF6C00"); tree_est.bind("<Button-1>", lambda e: toggle_checkbox(e, tree_est))
    def imprimir_estoque():
        conn = sqlite3.connect("sistema.db"); cursor = conn.cursor()
        cursor.execute("SELECT id, nome, preco, estoque FROM produtos")
        produtos = cursor.fetchall(); conn.close()
        if not produtos: messagebox.showinfo("Aviso", "Não há produtos no estoque para imprimir."); return
        f = GeradorPDF.gerar_relatorio_estoque(produtos)
        messagebox.showinfo("Sucesso", f"Relatório de estoque gerado: {f}")

    frame_botoes = ttk.Frame(main_frame); frame_botoes.pack(pady=10)
    ttk.Button(frame_botoes, text="Alterar Estoque", command=alterar_estoque_selecionados).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="Imprimir Estoque", command=imprimir_estoque).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="Editar Produto", command=editar_produto_selecionado).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="Remover Selecionados", command=remover_produtos_selecionados).pack(side="left", padx=5)
    ttk.Button(frame_botoes, text="Selecionar Todos", command=lambda: select_all(tree_est)).pack(side="left", padx=5)
    carregar_estoque()

def verificar_produtos_estoque_baixo():
    conn = sqlite3.connect("sistema.db"); cursor = conn.cursor(); cursor.execute("SELECT nome, estoque FROM produtos WHERE estoque <= ?", (LIMITE_ESTOQUE_BAIXO,)); produtos = cursor.fetchall(); conn.close(); return produtos

def mostrar_alerta_estoque(silencioso=False):
    produtos = verificar_produtos_estoque_baixo()
    if not produtos:
        if not silencioso: messagebox.showinfo("Estoque", "Todos os produtos estão com estoque em dia!")
        return
    msg = "Os seguintes produtos estão com estoque baixo:\n\n"
    for p in produtos: msg += f"• {p[0]}: {p[1]} unidades\n"
    if not silencioso: messagebox.showwarning("Alerta de Estoque", msg)

def atualizar_badge_alerta():
    produtos = verificar_produtos_estoque_baixo()
    if not produtos: label_badge_alerta.config(text="✅ Estoque OK", fg="#4CAF50")
    else: label_badge_alerta.config(text=f"⚠️ {len(produtos)} Alertas", fg="#FF6B35")

def tela_configuracoes():
    global dashboard_ativo; dashboard_ativo = False; limpar_tela()

    # Container principal com scroll caso a tela seja pequena
    canvas_cfg = tk.Canvas(main_frame, bg="#F0F0F0", highlightthickness=0)
    scrollbar_cfg = ttk.Scrollbar(main_frame, orient="vertical", command=canvas_cfg.yview)
    scrollable_frame = tk.Frame(canvas_cfg, bg="#F0F0F0")

    scrollable_frame.bind(
        "<Configure>",
        lambda e: canvas_cfg.configure(scrollregion=canvas_cfg.bbox("all"))
    )

    canvas_cfg.create_window((0, 0), window=scrollable_frame, anchor="nw")
    canvas_cfg.configure(yscrollcommand=scrollbar_cfg.set)

    canvas_cfg.pack(side="left", fill="both", expand=True)
    scrollbar_cfg.pack(side="right", fill="y")

    # Título
    ttk.Label(scrollable_frame, text="⚙️ Configurações do Sistema", font=("Segoe UI", 20, "bold")).pack(pady=20, padx=20, anchor="w")

    # Frame de Impressora
    frame_imp = ttk.LabelFrame(scrollable_frame, text=" Impressora Térmica (Recibos) "); frame_imp.pack(pady=10, padx=20, fill="x")

    config = ImpressoraTermica.carregar_config()

    # SELETOR DE IMPRESSORAS DO WINDOWS
    ttk.Label(frame_imp, text="1. Selecione a impressora na lista:", font=("Segoe UI", 10)).pack(pady=(15, 5), padx=15, anchor="w")

    lista_printers = ImpressoraTermica.listar_impressoras_windows()
    combo_printers = ttk.Combobox(frame_imp, values=lista_printers, state="readonly", font=("Segoe UI", 11))
    combo_printers.pack(pady=5, padx=15, fill="x")

    # Tenta selecionar a impressora salva
    if config.get("impressora") in lista_printers:
        combo_printers.set(config.get("impressora"))
    elif any("EPSON" in p.upper() for p in lista_printers):
        for p in lista_printers:
            if "EPSON" in p.upper():
                combo_printers.set(p); break

    # BOTÃO SALVAR EM DESTAQUE (Logo abaixo do seletor)
    def salvar():
        nome = combo_printers.get()
        if not nome:
            messagebox.showerror("Erro", "Por favor, selecione uma impressora na lista antes de salvar."); return
        if ImpressoraTermica.salvar_config(nome):
            messagebox.showinfo("Sucesso", f"Configuração Salva!\n\nImpressora definida: {nome}\n\nO sistema agora usará esta impressora para todos os recibos.")
        else:
            messagebox.showerror("Erro", "Não foi possível salvar.\nVerifique se o programa tem permissão para criar arquivos na pasta.")

    btn_salvar = tk.Button(frame_imp, text="💾 SALVAR CONFIGURAÇÃO", command=salvar, bg="#4CAF50", fg="white", font=("Segoe UI", 11, "bold"), pady=10, cursor="hand2")
    btn_salvar.pack(pady=20, padx=15, fill="x")

    # Outras opções
    ttk.Separator(frame_imp, orient="horizontal").pack(fill="x", padx=15, pady=10)

    def atualizar_lista():
        printers = ImpressoraTermica.listar_impressoras_windows()
        combo_printers["values"] = printers
        messagebox.showinfo("Impressoras", f"Lista atualizada! {len(printers)} impressoras encontradas.")

    ttk.Button(frame_imp, text="🔄 Atualizar Lista de Impressoras", command=atualizar_lista).pack(pady=10, padx=15, anchor="w")

    if not HAS_WIN32PRINT:
        tk.Label(frame_imp, text="⚠️ AVISO: Biblioteca de impressão não detectada.\nInstale com: pip install pywin32", fg="red", font=("Segoe UI", 9, "bold")).pack(pady=10)

def backup_banco_dados():
    try:
        if not os.path.exists("backups"):
            os.makedirs("backups")
        data_hora = datetime.now().strftime("%Y%m%d_%H%M%S")
        shutil.copy2("sistema.db", f"backups/sistema_backup_{data_hora}.db")
        # Manter apenas os últimos 10 backups para não lotar o disco
        backups = sorted([f for f in os.listdir("backups") if f.endswith(".db")])
        if len(backups) > 10:
            for i in range(len(backups) - 10):
                os.remove(os.path.join("backups", backups[i]))
    except Exception as e:
        print(f"Erro ao realizar backup: {e}")

def sair_programa():
    if messagebox.askyesno("Sair", "Deseja realmente sair?"):
        backup_banco_dados() # Realiza backup automático ao sair
        root.destroy()

def iniciar_sistema():
    criar_banco()
    login_window = tk.Tk()
    def on_login_success():
        login_window.destroy()
        global root, main_frame, label_badge_alerta
        root = tk.Tk(); root.title(f"ROTA DOS DRINKS - {USUARIO_LOGADO.upper()} ({CARGO_LOGADO})"); root.attributes("-fullscreen", True)
        menu_frame = tk.Frame(root, bg="#333333", width=160); menu_frame.pack(side="left", fill="y")

        # Menu lateral condicional
        if CARGO_LOGADO == 'admin':
            ttk.Button(menu_frame, text="💻 Dashboard", command=tela_dashboard).pack(pady=8, padx=5, fill="x")

        ttk.Button(menu_frame, text="💰 Venda", command=tela_venda).pack(pady=8, padx=5, fill="x")
        ttk.Button(menu_frame, text="💸 Controle Fiados", command=tela_debitos).pack(pady=8, padx=5, fill="x")
        ttk.Button(menu_frame, text="🗄️ Estoque", command=tela_estoque).pack(pady=8, padx=5, fill="x")
        ttk.Button(menu_frame, text="🪪 Cadastro", command=tela_cadastro).pack(pady=8, padx=5, fill="x")
        ttk.Button(menu_frame, text="⚙️ Configurações", command=tela_configuracoes).pack(pady=8, padx=5, fill="x")

        if CARGO_LOGADO == 'admin':
            tk.Label(menu_frame, text="── ADMIN ──", bg="#333333", fg="#FFD700", font=("Segoe UI", 9, "bold")).pack(pady=(10, 2))
            ttk.Button(menu_frame, text="📊 Faturamento", command=tela_faturamento).pack(pady=5, padx=5, fill="x")
            ttk.Button(menu_frame, text="📈 Histórico", command=tela_historico).pack(pady=5, padx=5, fill="x")
            ttk.Button(menu_frame, text="📋 Relatório + Vendidos", command=tela_relatorio_mais_vendidos).pack(pady=5, padx=5, fill="x")
            ttk.Button(menu_frame, text="👤 Usuários", command=tela_gerenciar_usuarios).pack(pady=5, padx=5, fill="x")

        tk.Label(menu_frame, text="─────────────", bg="#333333", fg="#555555").pack(pady=2)
        label_badge_alerta = tk.Label(menu_frame, text="✅ Estoque OK", bg="#333333", fg="#4CAF50", font=("Segoe UI", 9, "bold"), cursor="hand2"); label_badge_alerta.pack(pady=4, padx=5); label_badge_alerta.bind("<Button-1>", lambda e: mostrar_alerta_estoque())
        ttk.Button(menu_frame, text="Sair", command=sair_programa).pack(side="bottom", pady=20, padx=5, fill="x")

        main_frame = tk.Frame(root, bg="#F0F0F0"); main_frame.pack(side="right", fill="both", expand=True)

        # Define a tela inicial baseada no cargo
        if CARGO_LOGADO == 'admin':
            tela_dashboard()
        else:
            tela_venda()

        root.mainloop()
    LoginApp(login_window, on_login_success); login_window.mainloop()

if __name__ == "__main__":
    iniciar_sistema()
