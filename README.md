# meu-catalogo
Projeto para catálogo web gerenciável via GitHub com menu interativo para gestão do repositório. 

**Funcionalidades:**

- Atualizar o repositório local (`git pull`)
- Gerar o `index.html` a partir da estrutura de pastas
- Publicar as alterações (`git add`, `commit`, `push`)
- Sair

Você roda o script no seu computador, ele lê a estrutura de pastas do próprio repositório local e gera o catálogo estático. Depois, com o menu, você envia para o GitHub e ativa o GitHub Pages.

## 🧱 Estrutura do Repositório

Crie um repositório no seu GitHub (ex: `meu-catalogo`). Dentro dele, a estrutura deve ser:

```
meu-catalogo/
├── produtos/                  # Pasta raiz com os produtos
│   ├── jarra-ceramica/        # Pasta de um produto
│   │   ├── foto1.jpg
│   │   ├── foto2.png
│   │   └── descricao.txt      # Texto descritivo
│   ├── cachecol-lã/           # Outro produto
│   │   ├── principal.jpg
│   │   └── descricao.txt
│   └── ... (mais produtos)
├── config.txt                 # Opcional: dados do site (whatsapp, título, texto)
├── index.html                 # Será gerado pelo script
└── gerar_catalogo.py          # O script que vamos criar
```

> **Arquivo `config.txt`** (opcional, mas recomendado)  
> Para não precisar editar o script toda vez, você pode criar um arquivo `config.txt` com chaves e valores, assim:  
> ```
> titulo=Artesanato da Maria
> whatsapp=5511999999999
> texto_apresentacao=Peças únicas feitas com amor
> formulario_link=https://forms.gle/xxxxx
> ```

Se não existir, o script usa valores padrão.

**Cada produto**: uma subpasta em `produtos/`. Dentro dela, todas as imagens (`.jpg`, `.jpeg`, `.png`, `.gif`) e um arquivo obrigatório `descricao.txt` com a descrição em texto puro. A primeira imagem (ordem alfabética) será a miniatura do card.

## 🐍 Script Python (completo)

Crie um arquivo chamado `gerar_catalogo.py` no mesmo diretório do repositório (ao lado da pasta `produtos/`). Copie e cole o código abaixo.

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Gerador de Catálogo Estático para GitHub Pages
Leva a estrutura de pastas e gera index.html, com menu interativo para Git.
"""

import os
import sys
import subprocess
import glob
import webbrowser
from datetime import datetime

# ========== CONFIGURAÇÕES ==========
PASTA_PRODUTOS = "produtos"
ARQUIVO_CONFIG = "config.txt"
HTML_SAIDA = "index.html"

# Valores padrão (usados se não houver config.txt)
DEFAULT_TITULO = "Meu Catálogo de Artesanato"
DEFAULT_WHATSAPP = ""          # deixe vazio para não exibir botão
DEFAULT_TEXTO_APRES = "Bem-vindos! Conheça nossas peças artesanais."
DEFAULT_FORMULARIO = ""        # link opcional

# ========== FUNÇÕES DE LEITURA ==========
def carregar_config():
    """Lê config.txt e retorna um dicionário."""
    config = {
        "titulo": DEFAULT_TITULO,
        "whatsapp": DEFAULT_WHATSAPP,
        "texto_apresentacao": DEFAULT_TEXTO_APRES,
        "formulario_link": DEFAULT_FORMULARIO
    }
    if os.path.exists(ARQUIVO_CONFIG):
        with open(ARQUIVO_CONFIG, "r", encoding="utf-8") as f:
            for linha in f:
                linha = linha.strip()
                if not linha or linha.startswith("#"):
                    continue
                if "=" in linha:
                    chave, valor = linha.split("=", 1)
                    chave = chave.strip().lower()
                    valor = valor.strip()
                    if chave in config:
                        config[chave] = valor
    return config

def listar_produtos():
    """Retorna lista de produtos, cada um com nome, descrição, lista de imagens."""
    produtos = []
    if not os.path.isdir(PASTA_PRODUTOS):
        print(f"⚠️ Pasta '{PASTA_PRODUTOS}' não encontrada. Criando...")
        os.makedirs(PASTA_PRODUTOS)
        return []

    for nome_pasta in sorted(os.listdir(PASTA_PRODUTOS)):
        caminho_pasta = os.path.join(PASTA_PRODUTOS, nome_pasta)
        if not os.path.isdir(caminho_pasta):
            continue

        # Busca descrição
        descricao = ""
        caminho_desc = os.path.join(caminho_pasta, "descricao.txt")
        if os.path.exists(caminho_desc):
            with open(caminho_desc, "r", encoding="utf-8") as f:
                descricao = f.read().strip()

        # Busca imagens
        imagens = []
        extensoes = ("*.jpg", "*.jpeg", "*.png", "*.gif", "*.webp")
        for ext in extensoes:
            for img_path in glob.glob(os.path.join(caminho_pasta, ext)):
                imagens.append(img_path)
            for img_path in glob.glob(os.path.join(caminho_pasta, ext.upper())):
                imagens.append(img_path)
        imagens = sorted(set(imagens))   # remove duplicatas e ordena
        imagens_relativas = [os.path.relpath(img, start=".") for img in imagens]

        if not imagens_relativas:
            print(f"⚠️ Pasta '{nome_pasta}' sem imagens. Será ignorada.")
            continue

        produtos.append({
            "nome": nome_pasta,
            "descricao": descricao,
            "imagens": imagens_relativas,
            "imagem_principal": imagens_relativas[0]
        })
    return produtos

# ========== GERAÇÃO DO HTML ==========
def gerar_html(produtos, config):
    """Gera o arquivo index.html a partir dos produtos e configurações."""
    # Escapar dados para JavaScript (segurança)
    import json
    produtos_json = json.dumps(produtos, ensure_ascii=False)

    # Montar o HTML usando string multilinha
    html_template = f"""<!DOCTYPE html>
<html lang="pt-br">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{config['titulo']}</title>
    <style>
        /* Reset e estilos básicos */
        * {{ margin: 0; padding: 0; box-sizing: border-box; }}
        body {{
            font-family: system-ui, -apple-system, 'Segoe UI', Roboto, 'Helvetica Neue', sans-serif;
            background: #faf8f5;
            color: #2c2b28;
            line-height: 1.5;
            padding: 2rem 1rem;
        }}
        .container {{
            max-width: 1200px;
            margin: 0 auto;
        }}
        header {{
            text-align: center;
            margin-bottom: 3rem;
        }}
        h1 {{
            font-size: 2.5rem;
            color: #b45f2b;
            margin-bottom: 0.5rem;
        }}
        .whatsapp-btn {{
            background-color: #25D366;
            color: white;
            padding: 0.7rem 1.5rem;
            border-radius: 50px;
            text-decoration: none;
            font-weight: bold;
            display: inline-block;
            margin: 1rem 0;
            transition: 0.2s;
        }}
        .whatsapp-btn:hover {{ background-color: #128C7E; }}
        .apresentacao {{
            max-width: 700px;
            margin: 1.5rem auto;
            font-size: 1.1rem;
            color: #5e5b57;
        }}
        .produtos-grid {{
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
            gap: 2rem;
            margin-top: 2rem;
        }}
        .card {{
            background: white;
            border-radius: 1rem;
            overflow: hidden;
            box-shadow: 0 4px 12px rgba(0,0,0,0.05);
            transition: transform 0.2s, box-shadow 0.2s;
            cursor: pointer;
        }}
        .card:hover {{
            transform: translateY(-5px);
            box-shadow: 0 12px 24px rgba(0,0,0,0.1);
        }}
        .card-imagem {{
            width: 100%;
            height: 220px;
            object-fit: cover;
            background: #e6e0d8;
        }}
        .card-conteudo {{
            padding: 1.2rem;
        }}
        .card-titulo {{
            font-size: 1.3rem;
            margin-bottom: 0.5rem;
            color: #b45f2b;
        }}
        .card-descricao {{
            color: #4a4a46;
            font-size: 0.9rem;
            margin-bottom: 1rem;
        }}
        .card-botao {{
            background: #f0ede8;
            border: none;
            padding: 0.4rem 1rem;
            border-radius: 2rem;
            font-size: 0.85rem;
            cursor: pointer;
        }}
        footer {{
            text-align: center;
            margin-top: 3rem;
            color: #8e8a84;
            font-size: 0.8rem;
        }}
        /* Modal */
        .modal {{
            display: none;
            position: fixed;
            top: 0; left: 0;
            width: 100%; height: 100%;
            background: rgba(0,0,0,0.85);
            z-index: 1000;
            justify-content: center;
            align-items: center;
            overflow-y: auto;
        }}
        .modal-conteudo {{
            max-width: 900px;
            width: 90%;
            background: white;
            border-radius: 1rem;
            padding: 1.8rem;
            position: relative;
            margin: 2rem auto;
        }}
        .fechar-modal {{
            position: absolute;
            top: 1rem;
            right: 1.5rem;
            font-size: 2rem;
            cursor: pointer;
            color: #aaa;
        }}
        .modal-titulo {{
            font-size: 1.8rem;
            margin-bottom: 1rem;
        }}
        .modal-imagens {{
            display: flex;
            flex-wrap: wrap;
            gap: 1rem;
            margin: 1rem 0;
        }}
        .modal-imagens img {{
            max-width: 140px;
            border-radius: 0.5rem;
            cursor: pointer;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }}
        .whatsapp-encomenda, .formulario-link {{
            display: inline-block;
            background: #25D366;
            color: white;
            padding: 0.5rem 1.2rem;
            border-radius: 2rem;
            text-decoration: none;
            margin-top: 1rem;
            margin-right: 0.8rem;
        }}
        .formulario-link {{
            background: #6c757d;
        }}
    </style>
</head>
<body>
<div class="container">
    <header>
        <h1>{config['titulo']}</h1>
        {f'<a class="whatsapp-btn" href="https://wa.me/{config["whatsapp"]}?text=Olá! Vi seu catálogo e tenho interesse." target="_blank">📱 Fale conosco pelo WhatsApp</a>' if config['whatsapp'] else ''}
        <div class="apresentacao">{config['texto_apresentacao']}</div>
    </header>

    <section>
        <h2 style="margin-bottom: 1rem;">🧶 Nossas peças</h2>
        <div class="produtos-grid" id="produtos-grid">
            <!-- Cards serão preenchidos via JS -->
        </div>
    </section>
    <footer>
        <p>© {datetime.now().year} {config['titulo']} - Feito à mão</p>
    </footer>
</div>

<!-- Modal -->
<div id="modal" class="modal">
    <div class="modal-conteudo">
        <span class="fechar-modal" onclick="fecharModal()">&times;</span>
        <div id="modal-body"></div>
    </div>
</div>

<script>
    const produtos = {produtos_json};
    const whatsappNum = "{config['whatsapp']}";
    const formularioLink = "{config['formulario_link']}";

    function gerarCards() {{
        const grid = document.getElementById('produtos-grid');
        grid.innerHTML = '';
        produtos.forEach((p, idx) => {{
            const card = document.createElement('div');
            card.className = 'card';
            card.onclick = () => abrirModal(idx);
            card.innerHTML = `
                <img class="card-imagem" src="${{p.imagem_principal}}" alt="${{p.nome}}">
                <div class="card-conteudo">
                    <h3 class="card-titulo">${{p.nome}}</h3>
                    <p class="card-descricao">${{p.descricao.substring(0, 90)}}${{p.descricao.length > 90 ? '...' : ''}}</p>
                    <button class="card-botao">Ver mais</button>
                </div>
            `;
            grid.appendChild(card);
        }});
    }}

    function abrirModal(idx) {{
        const p = produtos[idx];
        let imagensHtml = '<div class="modal-imagens">';
        p.imagens.forEach(img => {{
            imagensHtml += `<img src="${{img}}" onclick="window.open('${{img}}', '_blank')" title="Clique para ampliar">`;
        }});
        imagensHtml += '</div>';
        const msgWhats = `Olá! Vi o produto *${{p.nome}}* no seu catálogo e gostaria de encomendar.`;
        const whatsLink = `https://wa.me/${{whatsappNum}}?text=${{encodeURIComponent(msgWhats)}}`;
        let botoes = '';
        if (whatsappNum) botoes += `<a class="whatsapp-encomenda" href="${{whatsLink}}" target="_blank">📦 Encomendar via WhatsApp</a>`;
        if (formularioLink) botoes += `<a class="formulario-link" href="${{formularioLink}}" target="_blank">📝 Preencher formulário</a>`;
        
        document.getElementById('modal-body').innerHTML = `
            <h2 class="modal-titulo">${{p.nome}}</h2>
            ${{imagensHtml}}
            <p><strong>Descrição:</strong></p>
            <p style="white-space: pre-wrap;">${{p.descricao}}</p>
            ${{botoes}}
        `;
        document.getElementById('modal').style.display = 'flex';
    }}

    function fecharModal() {{
        document.getElementById('modal').style.display = 'none';
    }}
    window.onclick = function(event) {{
        const modal = document.getElementById('modal');
        if (event.target === modal) fecharModal();
    }};
    gerarCards();
</script>
</body>
</html>"""
    # Escreve o arquivo
    with open(HTML_SAIDA, "w", encoding="utf-8") as f:
        f.write(html_template)
    print(f"✅ Página gerada: {HTML_SAIDA}")

# ========== FUNÇÕES GIT (menu interativo) ==========
def executar_comando(cmd, cwd=None):
    """Executa comando no shell e retorna (sucesso, saída)."""
    try:
        resultado = subprocess.run(cmd, shell=True, cwd=cwd, capture_output=True, text=True)
        if resultado.returncode != 0:
            print(f"⚠️ Comando falhou: {cmd}\n{resultado.stderr}")
            return False, resultado.stderr
        return True, resultado.stdout
    except Exception as e:
        print(f"❌ Erro ao executar: {e}")
        return False, str(e)

def verificar_git():
    """Verifica se git está instalado e se o diretório é um repositório."""
    sucesso, _ = executar_comando("git --version")
    if not sucesso:
        print("❌ Git não encontrado. Instale o Git e tente novamente.")
        return False
    if not os.path.isdir(".git"):
        print("⚠️ O diretório atual não é um repositório Git inicializado.")
        return False
    return True

def git_pull():
    print("🔄 Atualizando repositório (git pull)...")
    ok, out = executar_comando("git pull")
    if ok:
        print(out)
    return ok

def git_push(mensagem="Atualização automática do catálogo"):
    print("📦 Publicando alterações...")
    executar_comando("git add .")
    executar_comando(f'git commit -m "{mensagem}"')
    ok, out = executar_comando("git push")
    if ok:
        print(out)
        print("✅ Publicado com sucesso!")
    else:
        print("❌ Falha no push. Verifique suas credenciais ou conexão.")
    return ok

def abrir_pagina():
    """Tenta abrir a página localmente ou no GitHub Pages."""
    if os.path.exists(HTML_SAIDA):
        print("🔍 Abrindo o index.html no navegador...")
        webbrowser.open(f"file://{os.path.abspath(HTML_SAIDA)}")
    else:
        print("Arquivo index.html não encontrado. Gere o catálogo primeiro.")

# ========== MENU PRINCIPAL ==========
def menu():
    print("\n" + "="*50)
    print("       📦 GERENCIADOR DE CATÁLOGO - GITHUB PAGES")
    print("="*50)
    print("1. Atualizar repositório (git pull)")
    print("2. Gerar catálogo (index.html)")
    print("3. Publicar alterações (git add/commit/push)")
    print("4. Visualizar página local")
    print("5. Sair")
    print("-"*50)

def main():
    # Verifica se estamos em um repositório git (opcional, mas recomendado)
    if not os.path.exists(".git"):
        print("⚠️ Atenção: Este diretório não parece ser um repositório Git.")
        print("Se você já clonou ou inicializou o git, ignore. Caso contrário,")
        print("considere executar 'git init' e configurar o remote.\n")

    while True:
        menu()
        opcao = input("Escolha uma opção: ").strip()
        if opcao == "1":
            if verificar_git():
                git_pull()
            else:
                print("Git não disponível. Pule esta etapa.")
        elif opcao == "2":
            config = carregar_config()
            produtos = listar_produtos()
            if not produtos:
                print("❌ Nenhum produto encontrado. Adicione pastas com imagens dentro de 'produtos/'.")
                continue
            print(f"📂 Encontrados {len(produtos)} produtos.")
            gerar_html(produtos, config)
        elif opcao == "3":
            if verificar_git():
                msg = input("Mensagem do commit (Enter para padrão): ").strip()
                if not msg:
                    msg = "Atualização automática do catálogo"
                git_push(msg)
            else:
                print("Operação cancelada. Configure o git.")
        elif opcao == "4":
            abrir_pagina()
        elif opcao == "5":
            print("Saindo... Tenha um ótimo dia!")
            break
        else:
            print("Opção inválida. Digite 1 a 5.")

if __name__ == "__main__":
    main()
```

## 🖥️ Como usar (Windows, Linux, Mac)

### 1. Pré-requisitos
- **Python 3.6+** instalado (verifique com `python --version` ou `python3 --version`)
- **Git** instalado e configurado (com credenciais, se for push)
- Uma conta no GitHub e um repositório criado (vazio ou com a estrutura acima)

### 2. Clonar o repositório
```bash
git clone https://github.com/seu-usuario/meu-catalogo.git
cd meu-catalogo
```

### 3. Criar a estrutura inicial
Crie a pasta `produtos/`, coloque dentro uma subpasta de exemplo com algumas imagens e um `descricao.txt`.  
Crie também o `config.txt` (opcional, mas útil):
```txt
titulo=Artesanato da Ana
whatsapp=5511988888888
texto_apresentacao=Peças exclusivas, feitas à mão com materiais naturais.
formulario_link=https://forms.gle/abc123
```

### 4. Salvar o script
Cole o código Python acima em um arquivo `gerar_catalogo.py` na raiz do repositório.

### 5. Executar o script
- **Windows**: clique duas vezes ou abra o terminal e digite `python gerar_catalogo.py`
- **Linux/Mac**: no terminal, `python3 gerar_catalogo.py`

### 6. Usar o menu interativo
- Opção **2** → gera o `index.html` a partir da pasta `produtos/`
- Opção **1** → puxa alterações do GitHub (se outros colaboradores atualizarem)
- Opção **3** → envia as alterações (incluindo novas fotos, descrições) para o GitHub
- Opção **4** → abre o catálogo local no navegador (teste visual)

### 7. Ativar GitHub Pages
No repositório remoto (GitHub), vá em **Settings > Pages** e escolha a branch `main` (ou `master`) e a pasta raiz (`/`). O site estará disponível em `https://seu-usuario.github.io/meu-catalogo/`.

Toda vez que você rodar a opção 3 (publicar), o GitHub Pages será atualizado automaticamente (pode levar 1-2 minutos).

## 🔁 Fluxo de trabalho diário

1. **Adicionar um novo produto**: crie uma subpasta dentro de `produtos/`, coloque as fotos e o arquivo `descricao.txt`.
2. **Rodar o script** → opção 2 (gerar HTML) → opção 3 (publicar).
3. Pronto! O site está atualizado.

## 📌 Vantagens do uso no GitHub Pages
- **Totalmente offline**: você gera o site localmente e só publica quando quiser.
- **Sem depender de terceiros**: o conteúdo é estático, rápido e pode ser versionado no Git.
- **Fácil backup**: todo o catálogo (fotos, descrições) fica no seu repositório GitHub.
- **Funciona em qualquer SO** (Python é multiplataforma).

Agora é só colocar a mão na massa e construir seu catálogo artesanal! 🧵✨
