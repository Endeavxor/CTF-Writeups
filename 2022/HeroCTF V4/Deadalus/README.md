# HeroCTF V4 - PROG - HARD - Auteur : Log_S


## I. Description 

Vous trouverez la description complète du challenge ici : challenge.md

## II. Rapide retour au monde des mathématiques 
Avant de plonger plus en détail sur la stratégie adoptée, je vais faire un rapide retour sur les notions nécessaires à sa compréhension.*(Vous pouvez passer à la suite si vous êtes déjà familier avec les graphes)*

### II.I Les graphes

Un **graphe** est simplement constitué d'un ensemble de **sommets** et **d'arêtes** *(qui indique quel sommet est relié à quel autre sommet)*

Les **graphes** sont des objets/structures mathématiques qui ont été profondément étudié aux travers des siècles et disposent donc de nombreux d'algorithmes pour résoudre des problèmes souvent complexes avec plus de facilité qu'un algorithme dans des paradigmes classiques.

Voici quelques exemples de graphes *(il en existe d'autres adapté à différents problèmes)*:

![]()

Le premier est le plus commun, le second est un graphe dit "étiqueté" et le dernier, qui va nous intéresser pour ce challenge, est un graphe dit **"orienté"**. En effet comme on peut le constater, ce ne sont **pas des arêtes** mais des **arcs**, rendant la liaison entre deux sommets **unidirectionnelle** et non bidirectionnelle comme c'était le cas auparavant.

### II.II Circuit dans un graphe orienté
> Wikipedia https://fr.wikipedia.org/wiki/Circuit_(th%C3%A9orie_des_graphes) 
>
> -- <cite>Dans un graphe orienté, on appelle circuit une suite d'arcs consécutifs (chemin) dont les deux sommets aux extrémités sont identiques</cite>

Pour illustrer, voici en rouge des circuits dans des graphes orientés :

![]()

Nous allons pour ce challenge nous intéresser à des circuits **élémentaires**, *correspondant aux boucles*, qui ne sont ni plus ni moins que des circuits dont chaque sommet n'apparaît qu'une seule fois *(ce qui n'est pas le cas par exemple d'un des circuits dans le second graphe : x2->x4->**x3**->x5->x6->**x3**->x2 )*

Et bien si vous avez compris ça, vous avez toutes les clés en main pour résoudre le challenge très facilement :)

*Pour ceux qui ne se seraient pas intéressé aux graphes, je vous conseille grandement d'y jeter un oeil car si vous arrivez à modéliser un problème sous forme de graphes, il a de forte chance qu'un algorithme puisse vous faciliter la vie pour sa résolution*

## III. Retour sur le challenge et explication de la stratégie
Laissons de côté les graphes un court instant et revenons sur notre problème : une fois arrivé sur une case qui impose une direction *(R,L,D ou U)*, nous devons suivre cette direction jusqu'à arriver sur un des quatre cas suivant : 

- Sur un point (.)
- Sur une nouvelle direction *(R,L,D ou U)*
- Hors du labyrithe
- Sur une case spéciale *( - ou | )* dont le traitement varie selon la direction empruntée et la case en question

Si l'on schématise ce parcours sur un des exemples voici ce que l'on a : 

![]()
![]()

Comme on peut le remarquer, les points *('.')* sont redondants et on pourrait simplement aller directement à la prochaine case. Si l'on connecte entre elles les cases sans passer par les points et qu'on retire aussi les cas où l'on est bloqué *(par exemple '|' ici)*, on obtient un graphe bien plus digeste : 

![]()

Et si vous ne voyez toujours pas où je veux en venir, retirons le labyrinthe : 

![]()

Eh bien nous voilà en présence d'un magnifique graphe composé de **2 circuits élémentaires** ! Exactement ce que nous recherchions.

Très bien cela saute aux yeux qu'il y a 2 circuits élémentaires, mais comment fait-on pour les trouver ? Et bien comme expliqué plus haut, les graphes ont été grandement étudié et il existe déjà des algorithmes qui permettent de trouver des circuits élémentaires dans un graphe orienté. Dans notre cas, l'algorithme présent dans la librairie que nous utiliserons sera celui de DONALD B. JOHNSON (https://www.cs.tufts.edu/comp/150GA/homeworks/hw1/Johnson%2075.PDF) disposant d'une complexité linéaire.

Grâce à la modélisation du problème sous forme de graphe, le challenge se simplifie en  : "**Représentez le labyrinthe sous forme de graphe**", bien plus simple non :) ?

## IV. Implémentation de la solution


```python3

from enum import Enum # Servira pour factoriser le code
from pwn import * # Facilite la communication avec le serveur
import networkx as nx # Implémente la notion de Graphe et ses algorithmes

DIRECTIONS_NODES = ['R','L','D','U']
SPECIALS_NODES = ['-','|']
conn = remote("172.17.0.2",7000)

class Direction(Enum):
    '''
    Classe de type Enum qui facilite la factorisation du code.

    Le déplacement dans le labyrinthe se faisant dans une seule direction et d'une seule case à la fois,
    chaque direction est composé d'un tuple : (déplacement ligne, déplacement colonne).
    '''
    RIGHT = (0,1)
    LEFT = (0,-1)
    DOWN = (1,0)
    UP = (-1,0)

def connectNode(currentRowIndex,currentColumnIndex,direction):
    '''
    Fonction permettant de connecter deux noeuds entre eux et de les ajouter au graphe G.

    @param: currentRowIndex L'index de la ligne de la case en cours d'analyse
    @param: currentRowIndex L'index de la colonne de la case en cours d'analyse
    @param: direction La direction pour trouver la prochaine case à lier à la case en cours d'analyse
    '''
    dRow,dCol = direction.value # Récupère le déplacement à faire sur les lignes et colonnes
    nextRowIndex = currentRowIndex + dRow # L'index de la ligne de la prochaine case 
    nextColumnIndex = currentColumnIndex +dCol # L'index de la colonne de la prochaine case 

    # Tant que la prochaine case n'est pas hors du labyrinthe
    while 0 <= nextRowIndex <= numberOfRows -1 and 0 <= nextColumnIndex <= numberOfColumns - 1:
        nextNode = maze[nextRowIndex][nextColumnIndex] # Récupère la prochaine case

        # 2 cases sont connectées entre-elles uniquement lorsque l'une de ces conditions est établie :
        if (nextNode in DIRECTIONS_NODES) or (nextNode == "-" and dRow !=0) or (nextNode == "|" and dCol!=0):

            # On connecte nos deux cases en les nommant par leurs positions dans le labyrinthe
            G.add_edge(str(currentRowIndex)+str(currentColumnIndex),str(nextRowIndex)+str(nextColumnIndex))
            
            if nextNode in SPECIALS_NODES:
                # La case spéciale sera traitée après
                specialNodes.add((nextRowIndex,nextColumnIndex))
            return
        
        if nextNode !=".":
            return

        # Va à la prochaine case 
        nextRowIndex += dRow
        nextColumnIndex += dCol

for i in range(16): # Il y a 16 labyrinthe à traiter
    G = nx.DiGraph() # Créer un graphe orienté (directed graph)
    maze = conn.recvuntil(b"Answer >> ")[:-len("Answer >> ")].strip().decode().split('\n')[1:] # Ligne très moche pour récupérer le labyrinthe du serveur
    
    numberOfRows = len(maze)
    numberOfColumns = len(maze[0])
    
    specialNodes = set() # Utilisation d'un set (aucune entrée dupliquée) pour stocker les cases spéciales (| or -)

    for rowIndex in range(numberOfRows):
        for colIndex in range(numberOfColumns):
            currentNode = maze[rowIndex][colIndex]
            if currentNode == 'R':
                connectNode(rowIndex,colIndex,Direction.RIGHT)
            elif currentNode == 'L':
                connectNode(rowIndex,colIndex,Direction.LEFT)
            elif currentNode == 'D':
                connectNode(rowIndex,colIndex,Direction.DOWN)
            elif currentNode == 'U':
                connectNode(rowIndex,colIndex,Direction.UP)

    for specialNodeCoordiantes in specialNodes.copy(): # .copy() est utilisée pour modifier dynamiquement le contenu du set()
        specialNodeRowIndex , specialNodeColIndex = specialNodeCoordiantes
        specialNode = maze[specialNodeRowIndex][specialNodeColIndex]
        
        if specialNode == "-": # Si cette case spéciale est atteinte, elle se connecte à ce qui se trouve sur sa gauche et sa droite
            connectNode(specialNodeRowIndex,specialNodeColIndex,Direction.LEFT)
            connectNode(specialNodeRowIndex,specialNodeColIndex,Direction.RIGHT)
        else: # Si cette case spéciale est atteinte, elle se connecte à ce qui se trouve au-dessus et en dessous
            connectNode(specialNodeRowIndex,specialNodeColIndex,Direction.UP)
            connectNode(specialNodeRowIndex,specialNodeColIndex,Direction.DOWN)

    conn.send(str(len(list(nx.simple_cycles(G)))).encode()+b'\n') # Ligne moche qui envoie le nombre de circuits élémentaires (boucles) au serveur

print(conn.recvuntil(b'}')) # Affiche le flag
conn.close()

```

```bash
python3 deadalus.py

[+] Opening connection to 172.17.0.2 on port 7000: Done
b'\nCongratz !\nHero{h0w_aM4ZEiNg_y0U_d1D_17_3v3n_beTt3R_th4n_4ri4dne}'
[*] Closed connection to 172.17.0.2 port 7000
```

