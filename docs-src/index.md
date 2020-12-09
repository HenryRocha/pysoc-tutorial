# Soc & python

- **Alunos:** Gustavo Braga, Henry Rocha e Thomas Queiroz
- **Curso:** Engenharia da Computação
- **Semestre:** 6
- **Contato:** thomas@turbano.com
- **Ano:** 2020

## Começando

Para seguir esse tutorial é necessário:

- **Hardware:** Raspberry Pi
- **Softwares:** Python 3.8
- **Documentos:** TODO

## Motivação

Os sistemas embarcados sao utilizados com muita frequencia no dia a dia para tarefas diversas. Para tarefas que exigem um alto processamento e alta performance, as linguagens que geralmente sao utilizadas sao as de baixo nivel, como por exemplo o C. Por estar mais proxima da linguagem de maquina, isso a torna muito rapida, mas tambem muito mais complexa de ser utilizada para desenvolvimentos grandes. Por sua vez, o python eh considerado uma das linguagens mais faceis e intuitivas de serem aprendidas. Porem, como o seu nivel eh alto, sua performance eh muito pior do que outras linguagens, o que a impossibilita de ser utilizada em sistemas embarcados que exigem alta performance, principalmente por conta de restricoes de hardware. 

Dito isso, seria muito bom conseguir o melhor dos dois mundos: uma rapida linguagem, que seja facil de escrever. Para isso, devemos montar um modulo em Python escrito em C, dessa forma podemos utiliza-lo e nao perder performance ao escrever python. Em outras palavras, podemos escrever programas complexos que irao rodar em sistemas embarcados em python, facilitando o desenvolvimento.

## Introdução

Existem diversos modos de extender as funcionalidades do Python. Um desses modos é criar uma biblioteca em C. Esse método leva a aumento de performance, além de possibilitar a chamada de funções e bibliotecas C, em um nível baixo.

Um modulo em python pode ser considerado como uma biblioteca, que pode ser importada e utilizada em codigo por um programa. Exemplos de modulos famosos em python sao math, openCV, numpy entre outros. O openCV por exemplo, exige muitas operacoes e poder computacional, por isso eh implementada em C.

Nesse tutorial, vamos ver como criar um módulo em C que implementa o Bubble Sort, e, posteriormente, comparar sua performance contra o memso algoritmo feito em Python na Raspberry Pi. O Bubble sort foi escolhido pois sua complexidade eh O(n^2) em seu pior caso, por isso a comparacao eh muito mais facil de ser visualizada.

## Montando um módulo

De modo a suportar extensões, o Python API defines um grupo de funções, macros e variáveis que proveem acesso a várias parte do _run-time_ do Python. O Python API é incorporado em um arquivo em C usando o _header_ "Python.h".

Vamos começar criando um novo arquivo em C.
```sh
touch pysoc_bubble_sort.c
```

Devemos então incluir as bibliotecas Python. Basta colocar essas duas linhas no começo do arquivo.
```c
// Configura o tamanho das variáveis para 
// usar "Py_ssize_t" ao invés de "int" do C.
#define PY_SSIZE_T_CLEAN
// Inclui a biblioteca do Python.
#include <Python.h>
```

Em seguida vamos definir a função responsável por realizar o Bubble Sort. 
Essa função é como qualquer outra função em C. Ela recebe um vetor (v) e o tamanho dele (n)
```c
static void bubble_sort(long v[], long n) {
    for (ssize_t i = n - 1; i > 0; i--) {
        bool swapped = false;

        for (ssize_t j = 0; j < i; j++) {
            if (v[j] > v[j + 1]) {
                long temp = v[j];
                v[j] = v[j + 1];
                v[j + 1] = temp;
                swapped = true;
            }
        }

        if (!swapped) {
            break;
        }
    }
}
```

Agora devemos criar uma função _wrapper_ para essa função. Essa será a função que pode ser chamada dentro do Python.
Para definir tal função, é necessário o uso de objetos especias, vindos do Python API.

!!! note
    Dentre os objetos especiais, o mais notável é o `PyObject`.
    
    Ele é uma estrutura de objeto usada para definir os tipos de objetos para o Python. Todos os objetos Python compartilham algumas informações que são definidas por essa estrutura e todos os outros objetos são uma extensão dela.
    
    Esse objeto diz ao interpretador do Python como ele deve tratar objetos e seus ponteiros. Por exemplo, configurar o retorno de uma função como um `PyObject` possibilita que o interpretador reconheca esse retorno um tipo de variável válido de Python.


Vamos começar definindo a função.

```c
PyObject *pysoc_bubble_sort(PyObject *self, PyObject *args) {
}
```

Aqui estamos criando uma função chamada `pysoc_bubble_sort`, que retorna um `PyObject` e tem dois parâmetros, `self` e `args`, ambos `PyObject`.

O argumento `self` aponta para o objeto do módulo para funções nível módulo. 
Caso essa função fosse um método de uma clase, apontaria para uma instância do objeto.


Já o argumento `args` é um ponteiro para uma tupla do Python, contendo todos os argumento da função. 
Todos os items da tupla são objetos Python e para que possam ser usados em nossa função C devemos convertê-los para valores C.
Para realizar essa conversão existe a função `PyArg_ParseTuple()`. Ela checa os tipos dos argumetos e os converte para valores C.


Como a função que vamos criar é o Bubble Sort, o único argumento que vamos receber será a própria lista de valores a serem ordenados.
Vamos então pegar o tamanho da lista passada, que é o primeiro argumento da função `bubble_sort` definida anteriormente.

```c
PyObject *pysoc_bubble_sort(PyObject *self, PyObject *args) {
    // Declarando a variável que guardará a lista de valores.
    PyObject *int_list;
    // Verificando se foi realmente passado uma lista como argumento
    // caso contrário, retorne NULL.
    if (!PyArg_ParseTuple(args, "O", &int_list)) return NULL;
    // Calculando o tamanho da lista passada.
    Py_ssize_t n = PyObject_Length(int_list);
    // Caso o tamanho seja negativo, retorne NULL.
    if (n == -1) return NULL;
}
```

Como a API do Python não disponibiliza uma função que converta uma lista de Python para um vetor em C, devemos fazer tal conversão manualmente.
O código a seguir realiza essa conversão.

```c
// Declarando o vetor em C. O tamanho foi obtido anteriormente.
long v[n];
// Loop que passa por todos os items da lista passada.
for (Py_ssize_t i = 0; i < n; i++) {
    // Pegando o item "i" da lista.
    PyObject *val = PyList_GetItem(int_list, i);
    // Transformando esse item em um "long" do C.
    long lval = PyLong_AsLong(val);
    // Caso esse valor seja negativo, ocorreu um erro e devemos retornar NULL.
    if (lval == -1) return NULL;
    // Inserir o item convertido na lista.
    v[i] = lval;
}
```

Agora que temos o tamanho do vetor (`n`) e o vetor em si (`v`) podemos chamar a funçào `bubble_sort`.

```c
bubble_sort(v, n);
```

```c
PyObject *pysoc_bubble_sort(PyObject *self, PyObject *args) {
    // FALTA EXPLICAR ESSA PARTE DO CÓDIGO. TODO
    PyObject *ret_list = PyList_New(n);

    for (Py_ssize_t i = 0; i < n; i++) {
        PyObject *pl = PyLong_FromLong(v[i]); // Py_BuildValue("i", v[i]);
        if (pl == NULL) return NULL;

        if (PyList_SetItem(ret_list, i, pl) == -1) return NULL;
        /* if (PyList_Append(ret_list, pl) == -1) return NULL; */
    }

    return ret_list;
}
```

## Instalando o módulo

Como usar o módulo criado, mkfile, etc.

## Rapidez do modulo - comparacao entre C e python

Para iniciar a comparacao, necessitamos de uma implementacao do bubble sort em python. Ela pode ser vista a seguir:

```py
from typing import List

def bubble_sort(v: List[int]) -> List[int]:
    n: int = len(v)

    for i in range(n - 1, 0, -1):
        swapped: bool = False
        for j in range(0, i):
            if v[j] > v[j + 1]:
                v[j], v[j + 1] = v[j + 1], v[j]
                swapped = True

        if not swapped:
            break
    return v
```

Com o bubble sort em python e o bubble sort em C, podemos comecar a comparacao. Configuracoes iniciais do arquivo:

```py
import time
import pysoc
import bubble
```

Utilizaremos agora a implementacao em python primeiro. Como pode ser visto abaixo, foi criada uma lista de 0 ate 10000 ao contrario, para podermos pegar o pior caso do algoritmo. Logo em seguida mrcamos o tempo inicial e chamamos a funcao. Assim que ele termina, marcamos o tempo final e printamos o tempo que o codigo levou para rodar.

```py
l = list(reversed(range(int(1e4))))
print("== Python ==")
time1 = time.time()
l1 = bubble.bubble_sort(l)
time0 = time.time()
print(f"> {time0 - time1} seconds")
```

Agora para nosso modulo implementado em C, faremos quase o mesmo codigo, que pode ser visto a seguir.

```py
print("== C ==")
time3 = time.time()
l2 = pysoc.bubble_sort(l)
time2 = time.time()
print(f"> {time2 - time3} seconds")
```
Juntando todo programa em um so:

```py
import time
import pysoc
import bubble

l = list(reversed(range(int(1e4))))
print("== Python ==")
time1 = time.time()
l1 = bubble.bubble_sort(l)
time0 = time.time()
print(f"> {time0 - time1} seconds")

print("== C ==")
time3 = time.time()
l2 = pysoc.bubble_sort(l)
time2 = time.time()
print(f"> {time2 - time3} seconds")
print(f"\n> C is {(time0 - time1)/(time2 - time3)} times faster")


print(l1 == l2 == sorted(l))
```

Se rodarmos o codigo, perceberemos que C eh 45.000 vezes mais rapido que python!

```sh
python3 comparacao.py
```

## Cross-Compile

Como fazer cross-compile do módulo feito em C.



TODO
- Explicar o .h
    * Número de argumentos
    * Help da funcao
    * Ponteiro para a funcao
- Explicar o setup.py
- Mudar a velocidade de python e C
- Instalação do modulo
- Cross compile ?
- Interação com GPIO ?