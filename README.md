 # ======================================================================
# conexao.py — Gerenciador de Conexão com o Neon (PostgreSQL)
# ======================================================================
# Eu sou o responsável por conectar ao banco na nuvem e garantir
# que as tabelas existam antes de qualquer operação.
# A senha NÃO fica aqui: eu leio a DATABASE_URL do arquivo .env.
# ======================================================================

import os
from pathlib import Path

import psycopg
from psycopg.rows import dict_row
from dotenv import load_dotenv

# ---------- CARREGAMENTO DO .env ----------
# Eu procuro o .env na raiz do projeto (a pasta acima de database/).
load_dotenv(Path(__file__).parent.parent / ".env")


def _url_do_banco():
    """
    Eu devolvo a connection string guardada no .env.

    Se ela não existir, eu aviso com uma mensagem clara
    em vez de deixar o erro aparecer lá na frente.
    """
    url = os.getenv("DATABASE_URL")
    if not url:
        raise RuntimeError(
            "DATABASE_URL não encontrada. Crie o arquivo .env na raiz "
            "do projeto com a connection string do Neon."
        )
    return url


def conectar():
    """
    Eu abro e retorno uma conexão com o Neon.

    Use sempre com 'with conectar() as conn:'. Ao sair do with,
    o psycopg confirma as alterações (commit) e fecha a conexão.
    Se der erro dentro do with, ele desfaz tudo (rollback).

    row_factory=dict_row faz cada linha vir como dicionário,
    igual ao sqlite3.Row dos capítulos anteriores: aluno["nome"].
    """
    return psycopg.connect(
        _url_do_banco(),
        row_factory=dict_row,
        connect_timeout=10,  # desisto depois de 10 s sem resposta
    )


def inicializar_banco():
    """
    Eu crio as tabelas do sistema, caso ainda não existam.

    Chamada uma vez, quando o programa abre (no main.py).
    """
    with conectar() as conn:
        # ---------- TABELA DE ALUNOS ----------
        # GENERATED ALWAYS AS IDENTITY é o jeito do PostgreSQL de
        # numerar os registros sozinho (no SQLite era AUTOINCREMENT).
        conn.execute("""
            CREATE TABLE IF NOT EXISTS alunos (
                id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                nome TEXT NOT NULL,
                idade INTEGER NOT NULL,
                turma TEXT NOT NULL
            )
        """)


def verificar_conexao():
    """
    Eu testo se o Neon está respondendo.

    Retorno True se conseguir conectar, False se não houver
    internet, se a senha estiver errada ou se faltar o .env.
    """
    try:
        with psycopg.connect(_url_do_banco(), connect_timeout=5) as conn:
            conn.execute("SELECT 1")
        return True
    except (psycopg.Error, RuntimeError):
        return False