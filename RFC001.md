# RFC 001 – Benchmark Adaptativo para Definição de Tempo por Linguagem no Juiz de Código

- *Implementation Owner:* Vandressa Galdino Soares  
- *Start Date:* 03-07-2025  
- *Status:* Draft  

## Resumo

Esta RFC propõe a implementação de um módulo de Benchmark Adaptativo no Juiz de Código, com o objetivo de calcular tempos-limite personalizados por linguagem de programação. A proposta busca garantir uma avaliação justa entre soluções escritas em C++ e Python, considerando o desempenho real em ambiente controlado.

## Declaração do Problema

Plataformas tradicionais de julgamento automatizado como Codeforces, UVA e VJudge utilizam soluções em C++ como base para definir o tempo-limite, aplicando um fator fixo (geralmente 2x ou 3x) para todos os códigos. Essa abordagem desconsidera diferenças naturais de desempenho entre linguagens, penalizando soluções corretas escritas em linguagens como Python, que possuem maior sobrecarga de execução, tempo de interpretação mais alto, e limites de pilha mais restritos.

Além disso, a falta de adaptação de tempo por linguagem leva a um cenário desmotivador no contexto educacional: alunos que implementam algoritmos corretos enfrentam falhas técnicas como Time Limit Exceeded (TLE) ou Stack Overflow, não por erro de lógica, mas pela linguagem escolhida. Isso prejudica a aprendizagem, especialmente entre iniciantes que têm mais afinidade com linguagens de alto nível como Python. 

Tais limitações dificultam a construção de avaliações justas, minam a confiança dos estudantes e forçam instrutores a evitarem problemas mais avançados ou técnicas como programação dinâmica e recursão profunda, que são justamente as mais ricas pedagogicamente.

## Y-Statement (Justificativa)
No contexto de um juiz de código voltado ao ensino de algoritmos,  
diante da necessidade de avaliar de forma justa soluções corretas escritas em linguagens mais lentas como Python,  
decidimos por um cálculo adaptativo de tempo-limite com base em benchmarks reais entre soluções equivalentes em C++ e Python,  
e rejeitamos o uso de multiplicadores fixos sobre o tempo da C++,  
para alcançar avaliações mais justas e melhor apoio pedagógico aos alunos que usam diferentes linguagens,  
aceitando que rotinas de benchmark precisarão ser executadas e armazenadas para cada problema, aumentando a complexidade inicial da configuração.

## Alternativas Consideradas e Rejeitadas

- *Fator fixo (2x ou 3x) sobre solução C++*: não representa a variação real entre linguagens e falha em capturar casos extremos como recursões profundas em Python.
- *Benchmark baseado somente na solução em Python*: não fornece uma referência sólida, pode resultar em tempos-limite superdimensionados e desestimular o uso de linguagens mais rápidas.
- *Tempo-limite padrão fixo por problema*: ignora o tipo de problema e a técnica usada, favorecendo heurísticas e abordagens subótimas.

## Proposta de Design

### Componente: Benchmark Engine

Um módulo chamado *Benchmark Engine* será responsável por:

1. Executar soluções de referência em C++ e Python para um problema específico;
2. Medir o tempo de execução no maior caso de teste disponível;
3. Calcular:
   - base_time_cpp: tempo real da solução em C++;
   - reference_python_time: tempo real da solução em Python;
   - adjustment_factor_python = reference_python_time / base_time_cpp;
4. Armazenar os dados em tabela benchmarks com problem_id, tempos e fator de ajuste.

### API

- POST /benchmark/{problem_id}
  - Requisição:
    json
    {
      "reference_cpp": "// código em C++",
      "reference_python": "# código em Python"
    }
    
  - Resposta esperada:
    json
    {
      "status": "benchmark_completed",
      "base_time_cpp": 0.76,
      "adjustment_factor_python": 2.5
    }
    

### Persistência

- Utiliza PostgreSQL via SQLAlchemy para armazenar:
  - problem_id
  - base_time_cpp
  - reference_python_time
  - adjustment_factor_python
  - Data do benchmark

### Reuso dos dados

Ao receber uma submissão de um aluno, o sistema busca o adjustment_factor correspondente à linguagem e aplica o tempo-limite:

python
adjusted_limit = base_time_cpp * adjustment_factor_python


### Exemplo prático

Dado um problema onde a solução em C++ levou *0.4s* e a solução equivalente em Python levou *1.6s*, o fator de ajuste será:


adjustment_factor_python = 1.6 / 0.4 = 4.0


Portanto, o tempo-limite para soluções em Python será 4 vezes o da solução base C++.

## Tecnologias Utilizadas e Justificativas

- *Docker*: utilizado para garantir execução isolada e replicável das soluções submetidas, evitando interferência do ambiente da máquina host. Permite simular um ambiente controlado de avaliação, essencial para comparar tempos entre linguagens.

- *Python e C++*: linguagens escolhidas para o MVP por representarem extremos de desempenho (alto nível vs. compilado), sendo também as mais utilizadas por alunos na disciplina. A comparação entre elas justifica o uso de benchmark adaptativo.

- *PostgreSQL com SQLAlchemy*: banco de dados relacional utilizado pela confiabilidade, robustez e boa integração com o ecossistema Python. O uso de ORM facilita a manutenção e clareza do código.

- *Flask*: framework leve para construção do backend da API REST, suficiente para a complexidade do MVP. Escolhido por sua simplicidade, boa documentação e integração com bibliotecas Python.

- *Docker Compose*: ferramenta utilizada para orquestrar os diferentes containers do sistema localmente, garantindo modularidade e independência entre os componentes.

Essas tecnologias foram escolhidas com base em critérios de simplicidade, portabilidade, maturidade e compatibilidade com o perfil educacional do projeto.

## Impactos Esperados

- Avaliações mais justas entre C++ e Python;
- Redução de falsos negativos (TLE injusto em Python);
- Reforço ao uso de técnicas mais ricas pedagogicamente (ex: recursão, programação dinâmica);
- Possibilidade de aplicar a mesma lógica para Java, JavaScript, etc. futuramente;
- Maior engajamento de alunos que preferem linguagens de alto nível;
- Pequeno custo de execução extra no momento de configuração do problema.

## Possibilidades Futuras

- Suporte a benchmark incremental para diferentes tamanhos de entrada (curva de crescimento);
- Geração automática de relatórios de performance para docentes;
- Extensão para classificação de complexidade empírica (O(n), O(n^2), etc.);
- Adição de linguagem R ou Go ao benchmarking.

## Questões em Aberto

- Qual estratégia de agregação usar: média, mediana ou percentil 90?
- Devemos recalcular os benchmarks periodicamente? Se sim, com que frequência?
- Como lidar com soluções Python com variabilidade alta de execução (ex: garbage collector)?
- Como prevenir benchmarks manipulados com soluções propositalmente lentas?
- Devemos limitar o fator de ajuste máximo para evitar tempos-limite excessivos?
- Como documentar e validar benchmarks em avaliações formais (ex: provas)?
- Como garantir consistência entre ambientes locais e servidores (variação de hardware)?

## Referências

- [RFC: Y-Statement Model](https://medium.com/olzzio/y-statements-10eb07b5a177)
- [Codeforces Time Limit Policy](https://codeforces.com/blog/entry/91363)
- [Vjudge FAQ](https://vjudge.net/faq)
