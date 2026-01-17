import json
import datetime
import random
import os
from pathlib import Path
from typing import Dict, List, Optional, Any
from dataclasses import dataclass, field, asdict
from enum import Enum
import hashlib


# ==================== ESTRUTURAS DE DADOS ====================
class TipoMemoria(Enum):
    PESSOA = "pessoa"
    EVENTO = "evento"
    OBJETO = "objeto"
    LUGAR = "lugar"
    PENSAMENTO = "pensamento"
    CONVERSA = "conversa"


class EstadoEmocional(Enum):
    NEUTRO = "neutro"
    SAUDADE = "saudade"
    GRATIDAO = "gratidao"
    TRISTEZA = "tristeza"
    PAZ = "paz"


@dataclass
class Memoria:
    """Uma memória estruturada com metadados"""
    id: str
    titulo: str
    conteudo: str
    tipo: TipoMemoria
    data_original: Optional[str] = None
    data_registro: datetime.datetime = field(default_factory=datetime.datetime.now)
    pessoas: List[str] = field(default_factory=list)
    lugares: List[str] = field(default_factory=list)
    objetos: List[str] = field(default_factory=list)
    sentimentos: List[str] = field(default_factory=list)
    tags: List[str] = field(default_factory=list)
    importancia: int = 3  # 1-5, onde 5 é mais importante
    privacidade: int = 1  # 1=público, 2=privado, 3=secreto
    
    def to_dict(self) -> Dict[str, Any]:
        return asdict(self)
    
    @property
    def resumo(self) -> str:
        """Retorna um resumo da memória"""
        return f"{self.titulo}: {self.conteudo[:100]}..."


@dataclass
class PerfilMemorial:
    """Perfil de uma pessoa sendo lembrada"""
    nome: str
    data_nascimento: Optional[str] = None
    data_falecimento: Optional[str] = None
    apelido: Optional[str] = None
    relacionamento: str = "indefinido"
    caracteristicas: List[str] = field(default_factory=list)
    foto_path: Optional[str] = None
    
    @property
    def idade(self) -> Optional[int]:
        """Calcula idade ou idade no falecimento"""
        if not self.data_nascimento:
            return None
        
        try:
            nasc = datetime.datetime.strptime(self.data_nascimento, "%d/%m/%Y")
            fim = datetime.datetime.strptime(self.data_falecimento, "%d/%m/%Y") if self.data_falecimento else datetime.datetime.now()
            
            anos = fim.year - nasc.year
            if (fim.month, fim.day) < (nasc.month, nasc.day):
                anos -= 1
            return anos
        except:
            return None


# ==================== SISTEMA PRINCIPAL ====================
class SistemaMemoriaViva:
    """
    Sistema terapêutico para preservação de memórias
    Funcionalidades:
    1. Cadastro estruturado de memórias
    2. Busca inteligente por contexto
    3. Lembretes afetivos
    4. Geração de histórias
    5. Exportação para diferentes formatos
    """
    
    def __init__(self, nome_usuario: str, pasta_dados: str = "memorias_vivas"):
        self.nome_usuario = nome_usuario
        self.pasta_dados = Path(pasta_dados)
        self.pasta_dados.mkdir(exist_ok=True)
        
        # Arquivos de dados
        self.arquivo_memorias = self.pasta_dados / "memorias.json"
        self.arquivo_perfis = self.pasta_dados / "perfis.json"
        self.arquivo_diario = self.pasta_dados / "diario_terapeutico.json"
        
        # Dados em memória
        self.memorias: Dict[str, Memoria] = {}
        self.perfis: Dict[str, PerfilMemorial] = {}
        self.diario: List[Dict] = []
        
        # Configurações
        self.estado_atual = EstadoEmocional.NEUTRO
        
        # Carrega dados existentes
        self.carregar_dados()
        
        # Se primeiro uso, cria dados iniciais
        if not self.memorias:
            self._criar_dados_iniciais()
    
    def carregar_dados(self):
        """Carrega todos os dados dos arquivos"""
        # Carrega memórias
        if self.arquivo_memorias.exists():
            try:
                with open(self.arquivo_memorias, 'r', encoding='utf-8') as f:
                    dados = json.load(f)
                    for id_memoria, dados_memoria in dados.items():
                        dados_memoria['tipo'] = TipoMemoria(dados_memoria['tipo'])
                        dados_memoria['data_registro'] = datetime.datetime.fromisoformat(
                            dados_memoria['data_registro']
                        )
                        self.memorias[id_memoria] = Memoria(**dados_memoria)
                print(f"✓ Carregadas {len(self.memorias)} memórias")
            except Exception as e:
                print(f"⚠ Erro ao carregar memórias: {e}")
        
        # Carrega perfis
        if self.arquivo_perfis.exists():
            try:
                with open(self.arquivo_perfis, 'r', encoding='utf-8') as f:
                    dados = json.load(f)
                    for nome, dados_perfil in dados.items():
                        self.perfis[nome] = PerfilMemorial(**dados_perfil)
                print(f"✓ Carregados {len(self.perfis)} perfis")
            except Exception as e:
                print(f"⚠ Erro ao carregar perfis: {e}")
    
    def salvar_dados(self):
        """Salva todos os dados nos arquivos"""
        try:
            # Salva memórias
            dados_memorias = {
                id_memoria: memoria.to_dict() 
                for id_memoria, memoria in self.memorias.items()
            }
            with open(self.arquivo_memorias, 'w', encoding='utf-8') as f:
                json.dump(dados_memorias, f, indent=2, ensure_ascii=False, default=str)
            
            # Salva perfis
            dados_perfis = {
                perfil.nome: asdict(perfil)
                for perfil in self.perfis.values()
            }
            with open(self.arquivo_perfis, 'w', encoding='utf-8') as f:
                json.dump(dados_perfis, f, indent=2, ensure_ascii=False)
            
            print("✓ Dados salvos com sucesso")
        except Exception as e:
            print(f"✗ Erro ao salvar dados: {e}")
    
    def _criar_dados_iniciais(self):
        """Cria alguns dados iniciais para demonstração"""
        print("\n✨ Criando dados iniciais de demonstração...")
        
        # Cria perfil exemplo
        perfil_exemplo = PerfilMemorial(
            nome="Olga Chrysostomo",
            data_nascimento="15/03/1945",
            data_falecimento="10/08/2010",
            relacionamento="mãe",
            caracteristicas=["carinhosa", "forte", "cozinheira excelente"],
            apelido="Mãezinha"
        )
        self.perfis[perfil_exemplo.nome] = perfil_exemplo
        
        # Cria algumas memórias de exemplo
        memorias_exemplo = [
            Memoria(
                id=self._gerar_id(),
                titulo="O Fusca barro",
                conteudo="O Fusca barro placa SS5147 que minha mãe dirigia. Ela sempre mantinha o carro impecável.",
                tipo=TipoMemoria.OBJETO,
                pessoas=["Olga Chrysostomo"],
                objetos=["Fusca", "carro"],
                sentimentos=["saudade", "carinho"],
                tags=["infância", "família"],
                importancia=4
            ),
            Memoria(
                id=self._gerar_id(),
                titulo="Robô de papelão aos 9 anos",
                conteudo="Meu primeiro robô feito com caixa de papelão, gravador de voz e fita K7.",
                tipo=TipoMemoria.EVENTO,
                pessoas=["Alexander Crisóstomo Dias"],
                objetos=["caixa de papelão", "gravador", "fita K7"],
                sentimentos=["orgulho", "criatividade"],
                tags=["infância", "criação"],
                importancia=5
            ),
            Memoria(
                id=self._gerar_id(),
                titulo="O livro Portas",
                conteudo="Meu livro publicado em 2011 pela editora Multifoco.",
                tipo=TipoMemoria.OBJETO,
                objetos=["livro", "publicação"],
                sentimentos=["realização", "orgulho"],
                tags=["literatura", "conquista"],
                importancia=4
            )
        ]
        
        for memoria in memorias_exemplo:
            self.memorias[memoria.id] = memoria
        
        self.salvar_dados()
        print("✓ Dados iniciais criados")
    
    def _gerar_id(self) -> str:
        """Gera um ID único para memórias"""
        timestamp = datetime.datetime.now().strftime("%Y%m%d%H%M%S")
        random_num = random.randint(1000, 9999)
        return f"mem_{timestamp}_{random_num}"
    
    # ==================== FUNÇÕES PRINCIPAIS ====================
    
    def adicionar_memoria(self):
        """Interface para adicionar uma nova memória"""
        print("\n" + "="*50)
        print("ADICIONAR NOVA MEMÓRIA")
        print("="*50)
        
        titulo = input("Título da memória: ").strip()
        if not titulo:
            print("✗ Título é obrigatório")
            return
        
        print("\nDescreva a memória (pode ser longo, digite 'fim' em uma linha vazia para terminar):")
        linhas = []
        while True:
            linha = input().strip()
            if linha.lower() == 'fim':
                break
            linhas.append(linha)
        
        conteudo = "\n".join(linhas)
        if not conteudo:
            print("✗ Conteúdo é obrigatório")
            return
        
        # Tipo da memória
        print("\nTipo de memória:")
        for i, tipo in enumerate(TipoMemoria, 1):
            print(f"{i}. {tipo.value}")
        
        try:
            tipo_escolha = int(input("Escolha (1-6): "))
            tipo = list(TipoMemoria)[tipo_escolha - 1]
        except:
            tipo = TipoMemoria.EVENTO
        
        # Pessoas envolvidas
        pessoas = input("\nPessoas envolvidas (separadas por vírgula): ").strip()
        pessoas_lista = [p.strip() for p in pessoas.split(",") if p.strip()]
        
        # Sentimentos
        sentimentos = input("Sentimentos relacionados (separados por vírgula): ").strip()
        sentimentos_lista = [s.strip() for s in sentimentos.split(",") if s.strip()]
        
        # Tags
        tags = input("Palavras-chave/tags (separadas por vírgula): ").strip()
        tags_lista = [t.strip() for t in tags.split(",") if t.strip()]
        
        # Importância
        try:
            importancia = int(input("Importância (1-5, onde 5 é mais importante): "))
            importancia = max(1, min(5, importancia))
        except:
            importancia = 3
        
        # Data original (se lembrar)
        data_original = input("Data original da memória (DD/MM/AAAA, opcional): ").strip()
        
        # Cria a memória
        memoria = Memoria(
            id=self._gerar_id(),
            titulo=titulo,
            conteudo=conteudo,
            tipo=tipo,
            data_original=data_original if data_original else None,
            pessoas=pessoas_lista,
            sentimentos=sentimentos_lista,
            tags=tags_lista,
            importancia=importancia
        )
        
        self.memorias[memoria.id] = memoria
        self.salvar_dados()
        
        print(f"\n✓ Memória '{titulo}' adicionada com sucesso!")
        print(f"ID: {memoria.id}")
    
    def buscar_memorias(self):
        """Busca memórias por diferentes critérios"""
        print("\n" + "="*50)
        print("BUSCAR MEMÓRIAS")
        print("="*50)
        print("1. Por palavra-chave")
        print("2. Por pessoa")
        print("3. Por sentimento")
        print("4. Por tipo")
        print("5. Por importância")
        print("6. Ver todas")
        
        try:
            opcao = int(input("\nEscolha uma opção: "))
        except:
            opcao = 1
        
        resultados = []
        
        if opcao == 1:
            termo = input("Digite a palavra-chave: ").lower()
            for memoria in self.memorias.values():
                if (termo in memoria.titulo.lower() or 
                    termo in memoria.conteudo.lower() or 
                    any(termo in tag.lower() for tag in memoria.tags)):
                    resultados.append(memoria)
        
        elif opcao == 2:
            pessoa = input("Digite o nome da pessoa: ").strip()
            for memoria in self.memorias.values():
                if any(pessoa.lower() in p.lower() for p in memoria.pessoas):
                    resultados.append(memoria)
        
        elif opcao == 3:
            sentimento = input("Digite o sentimento: ").strip()
            for memoria in self.memorias.values():
                if any(sentimento.lower() in s.lower() for s in memoria.sentimentos):
                    resultados.append(memoria)
        
        elif opcao == 4:
            print("Tipos disponíveis:")
            for i, tipo in enumerate(TipoMemoria, 1):
                print(f"{i}. {tipo.value}")
            try:
                tipo_idx = int(input("Escolha o tipo: ")) - 1
                tipo_escolhido = list(TipoMemoria)[tipo_idx]
                resultados = [m for m in self.memorias.values() if m.tipo == tipo_escolhido]
            except:
                print("Opção inválida")
                return
        
        elif opcao == 5:
            try:
                importancia = int(input("Importância mínima (1-5): "))
                resultados = [m for m in self.memorias.values() if m.importancia >= importancia]
            except:
                print("Valor inválido")
                return
        
        elif opcao == 6:
            resultados = list(self.memorias.values())
            resultados.sort(key=lambda x: x.data_registro, reverse=True)
        
        # Mostra resultados
        if not resultados:
            print("\nNenhuma memória encontrada.")
        else:
            print(f"\n📚 Encontradas {len(resultados)} memória(s):")
            print("-" * 60)
            
            for i, memoria in enumerate(resultados, 1):
                print(f"{i}. [{memoria.tipo.value.upper()}] {memoria.titulo}")
                print(f"   💭 {memoria.conteudo[:80]}...")
                if memoria.pessoas:
                    print(f"   👥 Pessoas: {', '.join(memoria.pessoas)}")
                if memoria.sentimentos:
                    print(f"   ❤️ Sentimentos: {', '.join(memoria.sentimentos)}")
                print(f"   ⭐ Importância: {'★' * memoria.importancia}")
                print(f"   📅 Registrada em: {memoria.data_registro.strftime('%d/%m/%Y')}")
                print("-" * 40)
            
            # Opção para ver detalhes de uma memória
            if resultados:
                try:
                    ver_detalhes = input("\nVer detalhes de alguma memória? (número ou 'n' para não): ")
                    if ver_detalhes.lower() != 'n':
                        idx = int(ver_detalhes) - 1
                        if 0 <= idx < len(resultados):
                            self._mostrar_detalhes_memoria(resultados[idx])
                except:
                    pass
    
    def _mostrar_detalhes_memoria(self, memoria: Memoria):
        """Mostra detalhes completos de uma memória"""
        print("\n" + "="*60)
        print(f"DETALHES DA MEMÓRIA")
        print("="*60)
        print(f"Título: {memoria.titulo}")
        print(f"Tipo: {memoria.tipo.value}")
        print(f"Data original: {memoria.data_original or 'Não especificada'}")
        print(f"Data registro: {memoria.data_registro.strftime('%d/%m/%Y %H:%M')}")
        print(f"Importância: {'★' * memoria.importancia}")
        
        if memoria.pessoas:
            print(f"\n👥 Pessoas envolvidas:")
            for pessoa in memoria.pessoas:
                print(f"  • {pessoa}")
        
        if memoria.sentimentos:
            print(f"\n❤️ Sentimentos associados:")
            for sentimento in memoria.sentimentos:
                print(f"  • {sentimento}")
        
        if memoria.tags:
            print(f"\n🏷️ Tags: {', '.join(memoria.tags)}")
        
        print(f"\n📖 Conteúdo completo:")
        print("-" * 40)
        print(memoria.conteudo)
        print("-" * 40)
    
    def gerenciar_perfis(self):
        """Gerencia perfis de pessoas"""
        print("\n" + "="*50)
        print("GERENCIAR PERFIS")
        print("="*50)
        
        if not self.perfis:
            print("Nenhum perfil cadastrado.")
            criar = input("Deseja criar um perfil agora? (s/n): ")
            if criar.lower() == 's':
                self._criar_perfil()
            return
        
        print("\nPerfis cadastrados:")
        for i, (nome, perfil) in enumerate(self.perfis.items(), 1):
            idade_info = f" ({perfil.idade} anos)" if perfil.idade else ""
            print(f"{i}. {nome}{idade_info} - {perfil.relacionamento}")
        
        print("\nOpções:")
        print("1. Ver detalhes de um perfil")
        print("2. Criar novo perfil")
        print("3. Editar perfil")
        print("4. Voltar")
        
        try:
            opcao = int(input("\nEscolha: "))
        except:
            return
        
        if opcao == 1:
            try:
                idx = int(input("Número do perfil: ")) - 1
                nome = list(self.perfis.keys())[idx]
                self._mostrar_detalhes_perfil(nome)
            except:
                print("Opção inválida")
        
        elif opcao == 2:
            self._criar_perfil()
        
        elif opcao == 3:
            try:
                idx = int(input("Número do perfil para editar: ")) - 1
                nome = list(self.perfis.keys())[idx]
                self._editar_perfil(nome)
            except:
                print("Opção inválida")
    
    def _criar_perfil(self):
        """Cria um novo perfil"""
        print("\n" + "-"*40)
        print("CRIAR NOVO PERFIL")
        
        nome = input("Nome completo: ").strip()
        if not nome:
            print("✗ Nome é obrigatório")
            return
        
        apelido = input("Apelido (opcional): ").strip()
        relacionamento = input("Relacionamento com você (ex: mãe, pai, amigo): ").strip()
        
        data_nascimento = input("Data de nascimento (DD/MM/AAAA, opcional): ").strip()
        data_falecimento = input("Data de falecimento (DD/MM/AAAA, opcional): ").strip()
        
        print("\nCaracterísticas (digite uma por linha, linha vazia para terminar):")
        caracteristicas = []
        while True:
            caract = input().strip()
            if not caract:
                break
            caracteristicas.append(caract)
        
        perfil = PerfilMemorial(
            nome=nome,
            apelido=apelido if apelido else None,
            relacionamento=relacionamento,
            data_nascimento=data_nascimento if data_nascimento else None,
            data_falecimento=data_falecimento if data_falecimento else None,
            caracteristicas=caracteristicas
        )
        
        self.perfis[nome] = perfil
        self.salvar_dados()
        print(f"✓ Perfil de '{nome}' criado com sucesso!")
    
    def _mostrar_detalhes_perfil(self, nome: str):
        """Mostra detalhes de um perfil"""
        if nome not in self.perfis:
            print("Perfil não encontrado")
            return
        
        perfil = self.perfis[nome]
        
        print("\n" + "="*50)
        print(f"PERFIL: {perfil.nome}")
        print("="*50)
        
        if perfil.apelido:
            print(f"Apelido: {perfil.apelido}")
        
        print(f"Relacionamento: {perfil.relacionamento}")
        
        if perfil.data_nascimento:
            print(f"Data de nascimento: {perfil.data_nascimento}")
        
        if perfil.data_falecimento:
            print(f"Data de falecimento: {perfil.data_falecimento}")
        
        if perfil.idade:
            print(f"Idade: {perfil.idade} anos")
        
        if perfil.caracteristicas:
            print("\nCaracterísticas marcantes:")
            for caract in perfil.caracteristicas:
                print(f"  • {caract}")
        
        # Mostra memórias relacionadas a esta pessoa
        memorias_relacionadas = [
            m for m in self.memorias.values() 
            if nome in m.pessoas
        ]
        
        if memorias_relacionadas:
            print(f"\n📚 {len(memorias_relacionadas)} memória(s) relacionada(s):")
            for memoria in memorias_relacionadas[:5]:  # Limita a 5
                print(f"  • {memoria.titulo}")
        
        print("\n" + "-"*50)
    
    def _editar_perfil(self, nome: str):
        """Edita um perfil existente"""
        if nome not in self.perfis:
            print("Perfil não encontrado")
            return
        
        perfil = self.perfis[nome]
        print(f"\nEditando perfil: {nome}")
        
        novo_apelido = input(f"Apelido [{perfil.apelido or 'nenhum'}]: ").strip()
        if novo_apelido:
            perfil.apelido = novo_apelido
        
        novo_relacionamento = input(f"Relacionamento [{perfil.relacionamento}]: ").strip()
        if novo_relacionamento:
            perfil.relacionamento = novo_relacionamento
        
        print("\nEditar características atuais:")
        for i, caract in enumerate(perfil.caracteristicas, 1):
            print(f"{i}. {caract}")
        
        print("\nOpções: (a) Adicionar, (r) Remover, (enter) Manter")
        opcao = input("Escolha: ").lower()
        
        if opcao == 'a':
            nova = input("Nova característica: ").strip()
            if nova:
                perfil.caracteristicas.append(nova)
        elif opcao == 'r':
            try:
                idx = int(input("Número para remover: ")) - 1
                if 0 <= idx < len(perfil.caracteristicas):
                    perfil.caracteristicas.pop(idx)
            except:
                print("Índice inválido")
        
        self.salvar_dados()
        print("✓ Perfil atualizado!")
    
    def gerar_relatorio(self):
        """Gera um relatório das memórias"""
        print("\n" + "="*50)
        print("GERAR RELATÓRIO")
        print("="*50)
        
        if not self.memorias:
            print("Nenhuma memória para gerar relatório.")
            return
        
        total = len(self.memorias)
        por_tipo = {}
        por_importancia = {1: 0, 2: 0, 3: 0, 4: 0, 5: 0}
        
        for memoria in self.memorias.values():
            # Conta por tipo
            tipo_nome = memoria.tipo.value
            por_tipo[tipo_nome] = por_tipo.get(tipo_nome, 0) + 1
            
            # Conta por importância
            por_importancia[memoria.importancia] += 1
        
        # Data da memória mais antiga
        datas = [m.data_registro for m in self.memorias.values()]
        mais_antiga = min(datas) if datas else None
        
        print(f"\n📊 RELATÓRIO DO SISTEMA")
        print(f"Memórias totais: {total}")
        
        if mais_antiga:
            print(f"Registrando memórias desde: {mais_antiga.strftime('%d/%m/%Y')}")
        
        print(f"\n📈 Distribuição por tipo:")
        for tipo, quantidade in sorted(por_tipo.items()):
            porcentagem = (quantidade / total) * 100
            print(f"  {tipo:15} {quantidade:3} ({porcentagem:.1f}%)")
        
        print(f"\n⭐ Distribuição por importância:")
        for importancia in range(5, 0, -1):
            quantidade = por_importancia[importancia]
            if quantidade > 0:
                porcentagem = (quantidade / total) * 100
                estrelas = "★" * importancia
                print(f"  {estrelas:5} {quantidade:3} ({porcentagem:.1f}%)")
        
        # Pessoas mais mencionadas
        todas_pessoas = []
        for memoria in self.memorias.values():
            todas_pessoas.extend(memoria.pessoas)
        
        if todas_pessoas:
            from collections import Counter
            contador = Counter(todas_pessoas)
            mais_comuns = contador.most_common(5)
            
            print(f"\n👥 Pessoas mais mencionadas:")
            for pessoa, quantidade in mais_comuns:
                print(f"  {pessoa:20} {quantidade:3} menções")
        
        # Exportar opção
        exportar = input("\nDeseja exportar este relatório? (s/n): ")
        if exportar.lower() == 's':
            self._exportar_relatorio(por_tipo, por_importancia, total)
    
    def _exportar_relatorio(self, por_tipo: Dict, por_importancia: Dict, total: int):
        """Exporta o relatório para um arquivo"""
        data_atual = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        nome_arquivo = self.pasta_dados / f"relatorio_{data_atual}.txt"
        
        try:
            with open(nome_arquivo, 'w', encoding='utf-8') as f:
                f.write("="*60 + "\n")
                f.write("RELATÓRIO DO SISTEMA MEMÓRIA VIVA\n")
                f.write(f"Gerado em: {datetime.datetime.now().strftime('%d/%m/%Y %H:%M:%S')}\n")
                f.write("="*60 + "\n\n")
                
                f.write(f"Memórias totais: {total}\n\n")
                
                f.write("Distribuição por tipo:\n")
                for tipo, quantidade in sorted(por_tipo.items()):
                    porcentagem = (quantidade / total) * 100
                    f.write(f"  {tipo:15} {quantidade:3} ({porcentagem:.1f}%)\n")
                
                f.write("\nDistribuição por importância:\n")
                for importancia in range(5, 0, -1):
                    quantidade = por_importancia[importancia]
                    if quantidade > 0:
                        porcentagem = (quantidade / total) * 100
                        estrelas = "★" * importancia
                        f.write(f"  {estrelas:5} {quantidade:3} ({porcentagem:.1f}%)\n")
                
                # Lista todas as memórias
                f.write("\n" + "="*60 + "\n")
                f.write("LISTA COMPLETA DE MEMÓRIAS\n")
                f.write("="*60 + "\n\n")
                
                for memoria in sorted(self.memorias.values(), 
                                    key=lambda x: x.data_registro, 
                                    reverse=True):
                    f.write(f"ID: {memoria.id}\n")
                    f.write(f"Título: {memoria.titulo}\n")
                    f.write(f"Tipo: {memoria.tipo.value}\n")
                    f.write(f"Data: {memoria.data_registro.strftime('%d/%m/%Y')}\n")
                    f.write(f"Importância: {'★' * memoria.importancia}\n")
                    
                    if memoria.pessoas:
                        f.write(f"Pessoas: {', '.join(memoria.pessoas)}\n")
                    
                    if memoria.sentimentos:
                        f.write(f"Sentimentos: {', '.join(memoria.sentimentos)}\n")
                    
                    f.write(f"Conteúdo: {memoria.conteudo[:200]}...\n")
                    f.write("-" * 40 + "\n")
            
            print(f"✓ Relatório exportado para: {nome_arquivo}")
        except Exception as e:
            print(f"✗ Erro ao exportar: {e}")
    
    def lembrete_afetivo(self):
        """Mostra um lembrete afetivo aleatório"""
        if not self.memorias:
            print("Nenhuma memória cadastrada ainda.")
            return
        
        memoria = random.choice(list(self.memorias.values()))
        
        print("\n" + "✨"*25)
        print("LEMBRETE AFETIVO")
        print("✨"*25)
        print(f"\n💭 {memoria.titulo}")
        print(f"\n📖 {memoria.conteudo[:200]}...")
        
        if memoria.pessoas:
            print(f"\n👥 Pessoas envolvidas: {', '.join(memoria.pessoas)}")
        
        if memoria.sentimentos:
            print(f"❤️  Sentimentos: {', '.join(memoria.sentimentos)}")
        
        print(f"\n⭐ Importância: {'★' * memoria.importancia}")
        print(f"📅 Registrada em: {memoria.data_registro.strftime('%d/%m/%Y')}")
        print("\n✨"*25)
    
    def criar_capsula_tempo(self):
        """Cria uma cápsula do tempo com memórias importantes"""
        print("\n" + "="*50)
        print("CRIAR CÁPSULA DO TEMPO")
        print("="*50)
        
        if not self.memorias:
            print("Nenhuma memória para criar cápsula.")
            return
        
        # Filtra memórias mais importantes
        memorias_importantes = [
            m for m in self.memorias.values() 
            if m.importancia >= 4  # Só as 4 e 5 estrelas
        ]
        
        if not memorias_importantes:
            print("Nenhuma memória com importância 4+ estrelas.")
            return
        
        print(f"\nSelecionando {len(memorias_importantes)} memória(s) importante(s)...")
        
        # Cria arquivo da cápsula
        data_atual = datetime.datetime.now()
        nome_capsula = self.pasta_dados / f"capsula_tempo_{data_atual.strftime('%Y%m%d')}.html"
        
        try:
            with open(nome_capsula, 'w', encoding='utf-8') as f:
                f.write("""<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cápsula do Tempo - Memória Viva</title>
    <style>
        body {
            font-family: 'Georgia', serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f9f7f4;
            color: #333;
            line-height: 1.6;
        }
        .header {
            text-align: center;
            border-bottom: 2px solid #8b7355;
            padding-bottom: 20px;
            margin-bottom: 40px;
        }
        .memoria {
            background: white;
            border-left: 4px solid #8b7355;
            padding: 20px;
            margin: 30px 0;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
        }
        .titulo {
            color: #8b7355;
            margin-top: 0;
        }
        .meta {
            font-size: 0.9em;
            color: #666;
            margin-bottom: 15px;
        }
        .importancia {
            color: #ff9900;
            font-size: 1.2em;
        }
        .footer {
            text-align: center;
            margin-top: 50px;
            padding-top: 20px;
            border-top: 1px solid #ddd;
            font-size: 0.9em;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="header">
        <h1>📜 Cápsula do Tempo</h1>
        <p>Memórias importantes preservadas digitalmente</p>
        <p><em>Criada em: """ + data_atual.strftime('%d/%m/%Y') + """</em></p>
    </div>
""")
                
                for memoria in memorias_importantes:
                    estrelas = "★" * memoria.importancia
                    f.write(f"""
    <div class="memoria">
        <h2 class="titulo">{memoria.titulo}</h2>
        <div class="meta">
            <span class="importancia">{estrelas}</span> | 
            Tipo: {memoria.tipo.value} | 
            Data: {memoria.data_registro.strftime('%d/%m/%Y')}
        </div>
        <p>{memoria.conteudo}</p>
""")
                    
                    if memoria.pessoas:
                        pessoas = ", ".join(memoria.pessoas)
                        f.write(f'        <p><strong>Pessoas:</strong> {pessoas}</p>\n')
                    
                    if memoria.sentimentos:
                        sentimentos = ", ".join(memoria.sentimentos)
                        f.write(f'        <p><strong>Sentimentos:</strong> {sentimentos}</p>\n')
                    
                    f.write("    </div>\n")
                
                f.write(f"""
    <div class="footer">
        <p>Esta cápsula contém {len(memorias_importantes)} memória(s) importante(s)</p>
        <p>Sistema Memória Viva • Preservando o que importa</p>
    </div>
</body>
</html>""")
            
            print(f"✓ Cápsula do tempo criada: {nome_capsula}")
            print("  Você pode abrir este arquivo HTML em qualquer navegador.")
            
        except Exception as e:
            print(f"✗ Erro ao criar cápsula: {e}")
    
    # ==================== MENU PRINCIPAL ====================
    
    def mostrar_menu(self):
        """Exibe o menu principal"""
        while True:
            print("\n" + "="*60)
            print("MEMÓRIA VIVA - Sistema Terapêutico de Preservação")
            print("="*60)
            print(f"👤 Usuário: {self.nome_usuario}")
            print(f"📚 Memórias: {len(self.memorias)} | 👥 Perfis: {len(self.perfis)}")
            print("-"*60)
            
            print("1. 📝 Adicionar nova memória")
            print("2. 🔍 Buscar memórias")
            print("3. 👤 Gerenciar perfis de pessoas")
            print("4. 📊 Gerar relatório")
            print("5. ✨ Lembrete afetivo aleatório")
            print("6. ⏳ Criar cápsula do tempo")
            print("7. 💾 Salvar dados")
            print("8. 🚪 Sair")
            
            try:
                opcao = int(input("\nEscolha uma opção (1-8): "))
            except ValueError:
                print("Por favor, digite um número válido.")
                continue
            
            if opcao == 1:
                self.adicionar_memoria()
            elif opcao == 2:
                self.buscar_memorias()
            elif opcao == 3:
                self.gerenciar_perfis()
            elif opcao == 4:
                self.gerar_relatorio()
            elif opcao == 5:
                self.lembrete_afetivo()
            elif opcao == 6:
                self.criar_capsula_tempo()
            elif opcao == 7:
                self.salvar_dados()
            elif opcao == 8:
                print("\n💝 Obrigado por preservar suas memórias.")
                print("Elas são importantes. Volte sempre.")
                break
            else:
                print("Opção inválida. Tente novamente.")


# ==================== INICIALIZAÇÃO ====================
def main():
    """Função principal do sistema"""
    print("\n" + "🌟"*30)
    print("MEMÓRIA VIVA - Sistema Terapêutico")
    print("🌟"*30)
    print("\nBem-vindo ao sistema de preservação de memórias.")
    print("Aqui você pode guardar e organizar lembranças importantes.")
    
    # Solicita nome do usuário
    nome_usuario = input("\nQual é o seu nome? ").strip()
    if not nome_usuario:
        nome_usuario = "Usuário"
    
    # Inicializa o sistema
    sistema = SistemaMemoriaViva(nome_usuario)
    
    # Mostra menu principal
    sistema.mostrar_menu()


if __name__ == "__main__":
    main()
