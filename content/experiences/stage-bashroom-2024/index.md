+++
title = "Stage Bashroom 2024"
+++

# Stage Bashroom 2024

{% alert(warning=true) %}
Ceci est un récapitulatif de mon stage 2024 au sein de Bashroom. J'explique ce que j'ai fait durant cette période, et j'illustre mes propos avec ce que j'ai écrit ou ce que j'ai résolu durant le stage.
{% end %}

## 1. Automatiser l'exportation de projet Godot

La première tâche que j'ai eu lors de mon stage fut de rendre automatique le processus d'exportation de projet du moteur godot.

### 1. Les templates d'exportation

J'ai commencé par utiliser Godot et exporter un projet depuis l'interface graphique du logiciel.

{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/godot-export-menu.png?raw=true", alt="Godot export menu", no_hover=true)}}

Le fait d'utiliser l'interface m'a permis de comprendre et d'identifier les différents paramètres importants à l'exportation d'un projet.

Notamment lors de la première exportation, si le logiciel ne détecte pas de template d'exportation alors il propose d'en télécharger un depuis le mirroir que nous voulons.

{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/godot-template-download1.png?raw=true", alt="Godot template download", no_hover=true)}}

{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/godot-template-download2.png?raw=true", alt="Godot template download", no_hover=true)}}

Suite à cette découverte, j'ai ajouté le code suivant dans mon script pour vérifier si un template de la bonne version existe déjà. Si ce n'est pas le cas, alors je la télécharge depuis github.

```bash
# Check if there are already export templates
echo "Check if there are export templates..."

mkdir -p ~/.local/share/godot/export_templates/$version

exportTpl=$(ls ~/.local/share/godot/export_templates/$version)

linkVersion=$(echo $version | grep -oP '(\d+(\.\d+)+)')


# Download export templates if there aren't already downloaded
if [[ $exportTpl == "" ]];
then
	echo "Export templates not found..."
	echo "Downloading export templates..."
	wget -q https://github.com/godotengine/godot/releases/download/$linkVersion-stable/Godot_v$linkVersion-stable_export_templates.tpz
	unzip -qq Godot_v$linkVersion-stable_export_templates.tpz
	mv templates/* ~/.local/share/godot/export_templates/$version
	rm -r templates
	rm Godot_v$linkVersion-stable_export_templates.tpz
else
	echo "Export templates found..."
fi
```

### 2. Les presets d'exportation

Ensuite j'ai suivit la documentation de godot pour l'utiliser en <abbr title="Command Line Interface">CLI</abbr>. J'ai trouvé des commandes qui s'avèrent être utile pour réaliser ma mission.

{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/godot-cli-doc.png?raw=true", alt="Godot CLI documentation", no_hover=true)}}

En observant les arguments, nous retrouvons <mark>preset</mark> et <mark>path</mark>. <strong>Path</strong> qui signifie tout simplement le chemin absolu ou relatif vers le projet godot, alors que <strong>Preset</strong> est un peu complexe à comprendre.

J'ai compris qu'ajouter un present d'exportation au projet comme sur l'image suivante :

{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/godot-export-menu.png?raw=true", alt="Godot adding export option menu", no_hover=true) }}

Permet de générer, si non existant, le fichier `export_presets.cfg` à la racine du projet Godot et ajoute la configuration du preset dedans avec les options que nous lui affectons.

Voici à quoi ressemble un preset dans le fichier `export_presets.cfg` :

```txt
[preset.0]

name="Web"
platform="Web"
runnable=true
dedicated_server=false
custom_features=""
export_filter="all_resources"
include_filter="*.csv"
exclude_filter=""
export_path="Exported_'$name'/'$name'.html"
encryption_include_filters=""
encryption_exclude_filters=""
encrypt_pck=false
encrypt_directory=false

[preset.0.options]

custom_template/debug=""
custom_template/release=""
variant/extensions_support=false
vram_texture_compression/for_desktop=true
vram_texture_compression/for_mobile=false
html/export_icon=true
html/custom_html_shell=""
html/head_include=""
html/canvas_resize_policy=2
html/focus_canvas_on_start=true
html/experimental_virtual_keyboard=false
progressive_web_app/enabled=false
progressive_web_app/offline_page=""
progressive_web_app/display=1
progressive_web_app/orientation=0
progressive_web_app/icon_144x144=""
progressive_web_app/icon_180x180=""
progressive_web_app/icon_512x512=""
progressive_web_app/background_color=Color(0, 0, 0, 1)
```

### 3. Hébergé le projet exporté

Ensuite pour ouvrir le projet exporter vers de l'HTML5, j'ai utilisé un serveur python pour avoir accès aux fichiers dans mon navigateur. Voici le code que j'ai ajouté à mon script pour créer un fichier .py afin d'avoir un serveur python pour hébergé le projet exporté :

```bash
# Create a python virtual environment to install needed libs
echo "Check if python venv is installed..."
if ! dpkg -l | grep -q python3-venv;
then
	echo "Installing python venv..."
	sudo apt install python3-venv -y
fi

python3 -m venv Exported_$name/venv

# Installing libs in the python virtual env
echo "Creating python virtual environment..."
source Exported_$name/venv/bin/activate

echo "Installing python libs..."
python3 -m pip install Flask &>/dev/null
python3 -m pip install requests &>/dev/null


# Create a python server to host godot
echo "Creating server to host godot..."

godotServer="from http.server import SimpleHTTPRequestHandler
from http.server import HTTPServer
from socketserver import ThreadingMixIn
import os

class CustomRequestHandler(SimpleHTTPRequestHandler):
    def end_headers(self):
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
        self.send_header('Access-Control-Allow-Headers', 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token')
        self.send_header('Cross-Origin-Embedder-Policy', 'require-corp')
        self.send_header('Cross-Origin-Opener-Policy', 'same-origin')
        super().end_headers()

    def translate_path(self, path):
        if path == '/':
            return os.path.join(os.getcwd(), '$name.html')
        return super().translate_path(path)

class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
    pass

def run(server_class=ThreadedHTTPServer, handler_class=CustomRequestHandler, port=8000):
    server_address = ('', port)
    httpd = server_class(server_address, handler_class)
    print(f'Starting server on port {port}...')
    httpd.serve_forever()

if __name__ == '__main__':
    run()"

echo -e "$godotServer" > Exported_$name/godotServer.py
```

Le début du code sert surtout à ne pas installer les librairies nécessaires pour faire un serveur avec python sur la machine hôte mais sur un environnement virtuel.

### 4. Accèder aux ressources externes

Il y a une subtilité dans la tâche que j'ai du réaliser, je devais faire un script pour rendre automatique l'exportation de projet godot vers le web. Cependant, le projet sur lequel je devais essayer mon script avait besoin d'un fichier csv externe au projet.

La solution que j'ai adopté pour récupérer les informations du csv est de l'hébergé sur le repo gitlab du script, et à l'aide d'un autre serveur python avec les autorisations <abbr title="Cross-Origin Resource Sharing">CORS</abbr> nécessaires pour y accèder.

```bash
# Creating a server to handle request for godot project
echo "Creating python server to make request..."

requestServer="from flask import Flask, request, jsonify, send_from_directory
import requests

app = Flask(__name__)

@app.route('/proxy', methods=['GET'])
def proxy():
    # Récupérez l'URL de la demande client
    url = request.args.get('url')

    # Faites une requête HTTP vers Gitlab
    response = requests.get(url)

    # Renvoyez la réponse du serveur Gitlab avec les en-têtes CORS appropriés
    response_headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
        'Access-Control-Allow-Headers': 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token',
        'Cross-Origin-Embedder-Policy': 'require-corp',
        'Cross-Origin-Opener-Policy': 'same-origin'
    }

    return (response.text, response.status_code, response_headers)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)"

echo -e "$requestServer" > Exported_$name/proxy_server.py
```

<details>
    <summary>Voir le script en entier</summary>

    #!/bin/bash

    scriptUsage () {
    	echo -e "Usage: $0 <godot_path> <export_preset>
    	Examples :
    		$0 godot Web
    		$0 /bin/godot Web
    		$0 Godot_v4.2.1.stable Web"
    }

    godot=$1

    if [[ $godot == "" ]];
    then
    	scriptUsage
    	exit 1
    fi


    preset=$2

    if [[ $preset == "" ]];
    then
    	scriptUsage
    	exit 2
    fi


    name=$(pwd | rev | cut -d'/' -f1 | rev)


    version=$($godot --version | cut -d "." -f 1,2,3,4 | sed 's/\.official//')

    if [[ $version == "" ]];
    then
    	echo "Wrong godot path!"
    	exit 5
    fi


    # Export preset to HTML5 (Web)
    echo "Setup export preset..."

    presetCfg='[preset.0]

    name="Web"
    platform="Web"
    runnable=true
    dedicated_server=false
    custom_features=""
    export_filter="all_resources"
    include_filter="*.csv"
    exclude_filter=""
    export_path="Exported_'$name'/'$name'.html"
    encryption_include_filters=""
    encryption_exclude_filters=""
    encrypt_pck=false
    encrypt_directory=false

    [preset.0.options]

    custom_template/debug=""
    custom_template/release=""
    variant/extensions_support=false
    vram_texture_compression/for_desktop=true
    vram_texture_compression/for_mobile=false
    html/export_icon=true
    html/custom_html_shell=""
    html/head_include=""
    html/canvas_resize_policy=2
    html/focus_canvas_on_start=true
    html/experimental_virtual_keyboard=false
    progressive_web_app/enabled=false
    progressive_web_app/offline_page=""
    progressive_web_app/display=1
    progressive_web_app/orientation=0
    progressive_web_app/icon_144x144=""
    progressive_web_app/icon_180x180=""
    progressive_web_app/icon_512x512=""
    progressive_web_app/background_color=Color(0, 0, 0, 1)'

    echo -e "$presetCfg" > export_presets.cfg


    # Check if there are already export templates
    echo "Check if there are export templates..."

    mkdir -p ~/.local/share/godot/export_templates/$version

    exportTpl=$(ls ~/.local/share/godot/export_templates/$version)

    linkVersion=$(echo $version | grep -oP '(\d+(\.\d+)+)')


    # Download export templates if there aren't already downloaded
    if [[ $exportTpl == "" ]];
    then
    	echo "Export templates not found..."
    	echo "Downloading export templates..."
    	wget -q https://github.com/godotengine/godot/releases/download/$linkVersion-stable/Godot_v$linkVersion-stable_export_templates.tpz
    	unzip -qq Godot_v$linkVersion-stable_export_templates.tpz
    	mv templates/* ~/.local/share/godot/export_templates/$version
    	rm -r templates
    	rm Godot_v$linkVersion-stable_export_templates.tpz
    else
    	echo "Export templates found..."
    fi

    mkdir -p Exported_$name

    echo "Exporting godot project..."
    $godot --headless --export-release $preset Exported_$name/$name.html &>/dev/null


    # Create a python virtual environment to install needed libs
    echo "Check if python venv is installed..."
    if ! dpkg -l | grep -q python3-venv;
    then
    	echo "Installing python venv..."
    	sudo apt install python3-venv -y
    fi

    python3 -m venv Exported_$name/venv

    # Installing libs in the python virtual env
    echo "Creating python virtual environment..."
    source Exported_$name/venv/bin/activate

    echo "Installing python libs..."
    python3 -m pip install Flask &>/dev/null
    python3 -m pip install requests &>/dev/null


    # Create a python server to host godot
    echo "Creating server to host godot..."

    godotServer="from http.server import SimpleHTTPRequestHandler
    from http.server import HTTPServer
    from socketserver import ThreadingMixIn
    import os

    class CustomRequestHandler(SimpleHTTPRequestHandler):
        def end_headers(self):
            self.send_header('Access-Control-Allow-Origin', '*')
            self.send_header('Access-Control-Allow-Methods', 'GET, POST, OPTIONS')
            self.send_header('Access-Control-Allow-Headers', 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token')
            self.send_header('Cross-Origin-Embedder-Policy', 'require-corp')
            self.send_header('Cross-Origin-Opener-Policy', 'same-origin')
            super().end_headers()

        def translate_path(self, path):
            if path == '/':
                return os.path.join(os.getcwd(), '$name.html')
            return super().translate_path(path)

    class ThreadedHTTPServer(ThreadingMixIn, HTTPServer):
        pass

    def run(server_class=ThreadedHTTPServer, handler_class=CustomRequestHandler, port=8000):
        server_address = ('', port)
        httpd = server_class(server_address, handler_class)
        print(f'Starting server on port {port}...')
        httpd.serve_forever()

    if __name__ == '__main__':
        run()"

    echo -e "$godotServer" > Exported_$name/godotServer.py


    # Creating a server to handle request for godot project
    echo "Creating python server to make request..."

    requestServer="from flask import Flask, request, jsonify, send_from_directory
    import requests

    app = Flask(__name__)

    @app.route('/proxy', methods=['GET'])
    def proxy():
        # Récupérez l'URL de la demande client
        url = request.args.get('url')

        # Faites une requête HTTP vers Gitlab
        response = requests.get(url)

        # Renvoyez la réponse du serveur Gitlab avec les en-têtes CORS appropriés
        response_headers = {
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
            'Access-Control-Allow-Headers': 'Origin, Accept, Content-Type, X-Requested-With, X-CSRF-Token',
            'Cross-Origin-Embedder-Policy': 'require-corp',
            'Cross-Origin-Opener-Policy': 'same-origin'
        }

        return (response.text, response.status_code, response_headers)

    if __name__ == '__main__':
        app.run(host='0.0.0.0', port=8080)"

    echo -e "$requestServer" > Exported_$name/proxy_server.py

    exit 0

</details>

## 2. Graphique D3.js pour afficher les futures formations

La deuxième tâche qui m'a été attribué est la suivante : faire un graphique avec D3.js pour afficher les futures sessions de formations de l'entreprise.

Il m'a été imposé d'utiliser [tree of life](https://observablehq.com/@d3/tree-of-life) pour réaliser ce graphique.

J'ai commencé par comprendre les concepts de D3.js et ensuite de comprendre le fonctionnement du tree of life.

### 1. Mise en place d'un serveur node.js

Pour affiché le graphique dans mon navigateur, j'ai mis en place un serveur avec node.js.

```js
import express from 'express';
const app = express()
const port = 3000

app.use(express.static('assets/html'));
app.use(express.static('assets/js'));
app.use(express.static('assets/json'));

app.get('/', (req, res) => {
  res.sendFile('index.html')
})

app.listen(port, () => {
  console.log(`Listening on port ${port}`)
})
```

L'objectif de ce serveur est juste de rendre accessible les ressources : html, js et autres fichiers externes utiles au graphique.

### 2. Création du graphique

Afin d'afficher le graphique, j'ai eu besoin d'un fichier html pour appeler le code javascript dans le navigateur.

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>d3js - Bashroom</title>

    <script type="module" src="index.js"></script>
</head>
<body>
</body>
</html>
```

Maintenant que tout était prêt pour afficher le grahique dans mon navigateur, il était temps de le réaliser.

Pour utiliser D3.js j'ai utiliser le <abbr title="Content Delivery Network">CDN</abbr> pour importer D3 dans mon script.

```js
import * as d3 from "https://cdn.jsdelivr.net/npm/d3@7/+esm";
```

Ensuite pour la simplicité, j'ai placé les données en brut dans le script. Voici un exemple de ce à quoi ressembler les données :

```js
let jsonData = [
    {
        "category": "front-end",
        "technos": [
            {
                "name": "angular",
                "difficulty": [
                    {
                        "name": "speed-run",
                        "duration": 1,
                        "dates": ["2024-01-08"]
                    },
                    {
                        "name": "basic",
                        "duration": 3,
                        "dates": ["2024-01-17", "2024-01-18", "2024-01-19"]
                    },
                    {
                        "name": "intermediate",
                        "duration": 4,
                        "dates": ["2024-02-05", "2024-02-06", "2024-02-07", "2024-02-08"]
                    },
                    {
                        "name": "advanced",
                        "duration": 5,
                        "dates": ["2024-02-26", "2024-02-27", "2024-02-28", "2024-02-29", "2024-02-30"]
                    }
                ]
            },
        ]
    }
];
```

C'est donc un tableau contenant des objets avec comme propriété `category` et `technos`, qui est un tableau contenant des objets pour définir la formation avec `name` et la difficulté défini par `difficulty` qui contient la difficulté, les dates et la durée des formations.

Ensuite il a fallu faire une fonction pour transformer ces données au format <abbr title="JavaScript Object Notation">JSON</abbr> en un format compréhensible pour D3.js.

La fonction s'appelle `transformData` et prend `inputData` en paramètre, qui correspond à nos données sous format <abbr title="JavaScript Object Notation">JSON</abbr>.

```js
function transformData(inputData) {
```

Nous commençons par initialiser `result` qui est un objet contenant le nom du noeud, un tableau contenant les noeuds enfant de ce noeud et collapsed pour savoir si le noeud en question est collapsed ou non.

{% alert(tip=true) %}
Il faut savoir qu'un tree of life est constitué de noeud (node) et de liens (links).
{% end %}

```js
const result = { "name": "Formations", "children": [], "collapsed": true };
```

Maintenant que notre format d'objet est défini avec notre premier noeud central s'appelant "Formations".

Nous pouvons itérer avec des boucles sur chacun des tableaux que nous avons dans nos données, donc `inputData`, `technos`, `difficulty` et `dates`. Cela nous permet d'ajouter en tant que noeud enfant chaque valeur à son noeud parent.

```js
    for (const categoryData of inputData) {
        const categoryNode = { "name": categoryData.category, "children": [], "collapsed": true };

        for (const techno of categoryData.technos) {
            const technoNode = { "name": techno.name, "children": [], "collapsed": true };

            for (const difficulty of techno.difficulty) {
                const difficultyNode = { "name": difficulty.name, "value": difficulty.duration, "children": [], "collapsed": true };

                for (const date of difficulty.dates) {
                    const dateNode = { "name": date };

                    difficultyNode.children.push(dateNode);
                }
                technoNode.children.push(difficultyNode);
            }

            categoryNode.children.push(technoNode);
        }

        result.children.push(categoryNode);
    }

    return result;
}
```

Nous initialisons ensuite deux variables pour stocker les données transformés. Une première constante qui contient le résultat de notre fonction `transformData` afin d'avoir accès aux données transformés intact. Puis une deuxième variable qui contient notre première valeur mais qui sera modifiable pour agir sur les noeuds.

```js
const globalData = transformData(jsonData);
let treeData = globalData;
```

La fonction `updateTree`, comme son nom l'indique, permet de mettre à jour l'abre du graphique. Cette fonction permet depuis les données de la variable `treeData` de généré les noeuds, les liens et les textes pour afficher le nom des noeuds. 

```js
function updateTree() {
    d3.select("body").select("svg").remove();

    // Constant for tree length
    const width = 928;
    const height = width;
    const cx = width * 0.5;
    const cy = height * 0.59;
    const radius = Math.min(width, height) / 2 - 120;

    // Tree instance with custom options and our data
    const tree = d3.tree()
        .size([2 * Math.PI, radius])
        .separation((a, b) => (a.parent == b.parent ? 1 : 2) / a.depth);

    const root = tree(d3.hierarchy(treeData)
        .sort((a, b) => d3.ascending(a.data.name, b.data.name)));

    const svg = d3.select("body")
        .append("svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [-cx, -cy, width, height])
        .attr("style", "width: 75%; height: auto; font: 10px sans-serif;");

    // Append links between each node
    svg.append("g")
        .attr("fill", "none")
        .attr("stroke", "#156")
        .attr("stroke-opacity", 0.4)
        .attr("stroke-width", 1.5)
        .selectAll()
        .data(root.links())
        .join("path")
        .attr("d", d3.linkRadial()
            .angle(d => d.x)
            .radius(d => d.y));

    // Append each node with mouse handlers
    svg.append("g")
        .selectAll()
        .data(root.descendants())
        .join("circle")
        .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0)`)
        .attr("fill", d => d.children ? "#555" : "#999")
        .attr("r", 2.5)
        .on("click", (e, d) => {
            d.data.collapsed ? expand(d.data) : collapse(d.data);
            updateTree();
        });

    // Append text for formation's name with mouse handlers
    svg.append("g")
        .attr("stroke-linejoin", "round")
        .attr("stroke-width", 3)
        .selectAll()
        .data(root.descendants())
        .join("text")
        .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0) rotate(${d.x >= Math.PI ? 180 : 0})`)
        .attr("dy", "0.31em")
        .attr("x", d => d.x < Math.PI === !d.children ? 6 : -6)
        .attr("text-anchor", d => d.x < Math.PI === !d.children ? "start" : "end")
        .attr("paint-order", "stroke")
        .attr("stroke", "white")
        .attr("fill", "currentColor")
        .text(d => d.data.name)
        .on("click", (e, d) => {
            d.data.collapsed ? expand(d.data) : collapse(d.data);
            updateTree();
        });
}
```

Après la fonction pour mettre à jour l'arbre, nous avons les deux fonctions permettant la fonctionnalité de développer les noeuds et de les réduire ou en anglais d'expand et de collapse. Elles permettent de mettre à jour l'arbre en modifiant les noeuds développés et réduits.

```js
function expand(d) {
    if (d._children) {
        d.children = d._children;
        d._children = null;
    } else if (d.children) {
        d._children = d.children;
        d.children = null;
    }
    if (d._children) d._children.forEach(expand);
}

function collapse(d) {
    if (d._children) {
        d.children = d._children;
        d._children = null;
    }
    if (d.children) d.children.forEach(collapse);
}
```

A la fin toute fin du script, nous trouvons une ligne pour appeler la fonction `updateTree` pour afficher pour la première fois le graphique.

```js
updateTree()
```
<details>
    <summary>Voir le script en entier</summary>

    import * as d3 from "https://cdn.jsdelivr.net/npm/d3@7/+esm";

    let jsonData = [
        {
            "category": "front-end",
            "technos": [
                {
                    "name": "angular",
                    "difficulty": [
                        {
                            "name": "speed-run",
                            "duration": 1,
                            "dates": ["2024-01-08"]
                        },
                        {
                            "name": "basic",
                            "duration": 3,
                            "dates": ["2024-01-17", "2024-01-18", "2024-01-19"]
                        },
                        {
                            "name": "intermediate",
                            "duration": 4,
                            "dates": ["2024-02-05", "2024-02-06", "2024-02-07", "2024-02-08"]
                        },
                        {
                            "name": "advanced",
                            "duration": 5,
                            "dates": ["2024-02-26", "2024-02-27", "2024-02-28", "2024-02-29", "2024-02-30"]
                        }
                    ]
                },
            ]
        }
    ];

    /**
     * Transform JSON data into an object for the tree
     * @param {JSON} inputData
     * @returns An object for the tree
     */
    function transformData(inputData) {
        const result = { "name": "Formations", "children": [], "collapsed": true };

        for (const categoryData of inputData) {
            const categoryNode = { "name": categoryData.category, "children": [], "collapsed": true };

            for (const techno of categoryData.technos) {
                const technoNode = { "name": techno.name, "children": [], "collapsed": true };

                for (const difficulty of techno.difficulty) {
                    const difficultyNode = { "name": difficulty.name, "value": difficulty.duration, "children": [], "collapsed": true };

                    for (const date of difficulty.dates) {
                        const dateNode = { "name": date };

                        difficultyNode.children.push(dateNode);
                    }
                    technoNode.children.push(difficultyNode);
                }

                categoryNode.children.push(technoNode);
            }

            result.children.push(categoryNode);
        }

        return result;
    }

    const globalData = transformData(jsonData);
    let treeData = globalData;

    function updateTree() {
        d3.select("body").select("svg").remove();

        // Constant for tree length
        const width = 928;
        const height = width;
        const cx = width * 0.5;
        const cy = height * 0.59;
        const radius = Math.min(width, height) / 2 - 120;

        // Tree instance with custom options and our data
        const tree = d3.tree()
            .size([2 * Math.PI, radius])
            .separation((a, b) => (a.parent == b.parent ? 1 : 2) / a.depth);

        const root = tree(d3.hierarchy(treeData)
            .sort((a, b) => d3.ascending(a.data.name, b.data.name)));

        const svg = d3.select("body")
            .append("svg")
            .attr("width", width)
            .attr("height", height)
            .attr("viewBox", [-cx, -cy, width, height])
            .attr("style", "width: 75%; height: auto; font: 10px sans-serif;");

        // Append links between each node
        svg.append("g")
            .attr("fill", "none")
            .attr("stroke", "#156")
            .attr("stroke-opacity", 0.4)
            .attr("stroke-width", 1.5)
            .selectAll()
            .data(root.links())
            .join("path")
            .attr("d", d3.linkRadial()
                .angle(d => d.x)
                .radius(d => d.y));

        // Append each node with mouse handlers
        svg.append("g")
            .selectAll()
            .data(root.descendants())
            .join("circle")
            .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0)`)
            .attr("fill", d => d.children ? "#555" : "#999")
            .attr("r", 2.5)
            .on("click", (e, d) => {
                d.data.collapsed ? expand(d.data) : collapse(d.data);
                updateTree();
            });

        // Append text for formation's name with mouse handlers
        svg.append("g")
            .attr("stroke-linejoin", "round")
            .attr("stroke-width", 3)
            .selectAll()
            .data(root.descendants())
            .join("text")
            .attr("transform", d => `rotate(${d.x * 180 / Math.PI - 90}) translate(${d.y},0) rotate(${d.x >= Math.PI ? 180 : 0})`)
            .attr("dy", "0.31em")
            .attr("x", d => d.x < Math.PI === !d.children ? 6 : -6)
            .attr("text-anchor", d => d.x < Math.PI === !d.children ? "start" : "end")
            .attr("paint-order", "stroke")
            .attr("stroke", "white")
            .attr("fill", "currentColor")
            .text(d => d.data.name)
            .on("click", (e, d) => {
                d.data.collapsed ? expand(d.data) : collapse(d.data);
                updateTree();
            });
    }

    function expand(d) {
        if (d._children) {
            d.children = d._children;
            d._children = null;
        } else if (d.children) {
            d._children = d.children;
            d.children = null;
        }
        if (d._children) d._children.forEach(expand);
    }

    function collapse(d) {
        if (d._children) {
            d.children = d._children;
            d._children = null;
        }
        if (d.children) d.children.forEach(collapse);
    }

    updateTree()
</details>

Voici le résultat de ce script dans le navigateur :
{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/d3-formations-result.png?raw=true", alt="Resultat script graphique D3.js", no_hover=true) }}

Nous pouvons voir que des noeuds sont collapse comme : nestjs, rust, typescript, ansible, aws et cloud. Nous pouvons aussi voir que des dates sont disponibles que pour les formations en bas à droite du graphique.

## 3. Graphique 2D avec D3.js et graphique 3D avec three.js pour la formation data science

La troisième tâche que j'ai eu à faire fut la suivante : faire un graphique 2D avec D3.js et un graphique 3D avec three.js pour la formation data science.

Contrairement à la deuxième tâche n'a pas été réalisé en local sur ma machine mais sur [Google Colab](https://colab.google/).

### 1. Graphique 2D avec D3.js

Pour le graphique 2D, j'ai du faire un [3D Force Graph](https://github.com/vasturiano/3d-force-graph) en 2D. Puisque c'est différent d'un tree of life, j'ai pas pu récupérer l'intégralité de ce que j'avais fait, seulement quelques parties similaires au deux format.

Nous retrouvons une fois de plus, le <abbr title="Content Delivery Network">CDN</abbr> pour importer D3.js.

```js
    import * as d3 from 'https://cdn.skypack.dev/d3@7';
```

Ensuite nous avons une variable `timeoutId` qui va nous permettre de mettre à jour automatiquement avec un intervalle de temps.

```js
    let timeoutId = undefined;
```

Nous retrouvons la fonction `draw`, qui s'apparente à la fonction `updateTree`, pour afficher le graphique sauf que cette fois-ci nous retrouvons également des fonctions dans cette fonction pour gérer les événements de drag and drop sur les noeuds.

```js
function draw({ source, data }) {
    // Define the container size
    const width = 800;
    const height = 600;

    d3.select("#visualization").select("svg").remove();

    // Create SVG container
    const svg = d3.select('#visualization')
        .append('svg')
        .attr('width', width)
        .attr('height', height);

    // Create a group for zooming
    const zoomGroup = svg.append('g');

    // Create force simulation
    const simulation = d3.forceSimulation(data.nodes)
        .force('link', d3.forceLink(data.links).id(d => d.id)) // Use d3.forceLink for the links
        .force('charge', d3.forceManyBody().strength(-50))
        .force('center', d3.forceCenter(width / 2, height / 2));

    // Append links
    const links = zoomGroup
        .selectAll('line')
        .data(data.links)
        .enter()
        .append('line')
        .attr('stroke', '#fff')
        .attr('stroke-width', 2);

    // Append group instead of node to add a text
    const node = zoomGroup
        .append("g")
        .attr("fill", "orange")
        .attr("stroke", "#fff")
        .attr("stroke-opacity", 1.5)
        .attr("stroke-width", 2)
        .selectAll("g")
        .data(data.nodes)
        .join("g")
        .call(drag(simulation));

    // Append circle to node "g"
    node
        .append('circle')
        .attr('r', 7.5);

    // Append text to node "g"
    node
        .append('text')
        .text(d => d.id)
        .attr("fill", "black")
        .attr("stroke", "none")
        .attr("font-size", "0.8em");

    // Add zoom behavior
    const zoom = d3.zoom()
        .scaleExtent([0.1, 4]) // Set the zoom scale range
        .on("zoom", (event) => {
            zoomGroup.attr("transform", event.transform);
        });

    // Apply the zoom behavior to the SVG
    svg.call(zoom);

    // Update simulation at each tick
    simulation.on('tick', () => {
        links
            .attr('x1', d => d.source.x)
            .attr('y1', d => d.source.y)
            .attr('x2', d => d.target.x)
            .attr('y2', d => d.target.y);

        node
            .attr('transform', d => `translate(${d.x},${d.y})`);
        });

    // Function to handle drags event
    function drag(simulation) {
        function dragstarted(event) {
            if (!event.active) simulation.alphaTarget(0.3).restart();
            event.subject.fx = event.subject.x;
            event.subject.fy = event.subject.y;
        }

        function dragged(event) {
            event.subject.fx = event.x;
            event.subject.fy = event.y;
        }

        function dragended(event) {
            if (!event.active) simulation.alphaTarget(0);
            event.subject.fx = null;
            event.subject.fy = null;
        }

        return d3.drag()
            .on("start", dragstarted)
            .on("drag", dragged)
            .on("end", dragended);
    }
}
```

Après la fonction `draw`, nous avons la fonction `dataFetch` qui permet de faire une requête sur cette route `http://localhost:8000/data` pour récupérer les données depuis un serveur local sur le Google Colab et mettre à jour le graphique. Tout en utilisant la variable `timeoutId` afin de définir l'intervalle de temps entre chaque appelle.

```js
function dataFetch({source}) {
    console.log('dataFetch', {source});
    if(timeoutId !== undefined){
        clearTimeout(timeoutId);
    }
    fetch('http://localhost:8000/data')
        .then(response => response.json())
        .then(data => {
            draw({source, data});
            timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
        })
        .catch(error => {
            console.error('Error:', error);
            timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
        });
}
```

Pour terminer, nous retrouvons un `addEventListener` sur un bouton pour gérer l'action du bouton et appeler la fonction `dataFetch`. Puis nous appelons `dataFetch` pour récupérer les données et afficher le graphique une première fois.

```js
document.getElementById('refreshButton').addEventListener(
    'click',
    () => dataFetch({source: 'refreshButton'})
);

dataFetch({source: 'initialization'});
```

<details>
    <summary>Voir le script en entier</summary>

    import * as d3 from 'https://cdn.skypack.dev/d3@7';

    const visualizationElement = document.getElementById('visualization');
    let timeoutId = undefined;

    function draw({ source, data }) {
      // Define the container size
      const width = 800;
      const height = 600;

      d3.select("#visualization").select("svg").remove();

      // Create SVG container
      const svg = d3.select('#visualization')
        .append('svg')
        .attr('width', width)
        .attr('height', height);

      // Create a group for zooming
      const zoomGroup = svg.append('g');

      // Create force simulation
      const simulation = d3.forceSimulation(data.nodes)
        .force('link', d3.forceLink(data.links).id(d => d.id)) // Use d3.forceLink for the links
        .force('charge', d3.forceManyBody().strength(-50))
        .force('center', d3.forceCenter(width / 2, height / 2));

      // Append links
      const links = zoomGroup
        .selectAll('line')
        .data(data.links)
        .enter()
        .append('line')
        .attr('stroke', '#fff')
        .attr('stroke-width', 2);

      // Append group instead of node to add a text
      const node = zoomGroup
        .append("g")
        .attr("fill", "orange")
        .attr("stroke", "#fff")
        .attr("stroke-opacity", 1.5)
        .attr("stroke-width", 2)
        .selectAll("g")
        .data(data.nodes)
        .join("g")
        .call(drag(simulation));

      // Append circle to node "g"
      node
        .append('circle')
        .attr('r', 7.5);

      // Append text to node "g"
      node
        .append('text')
        .text(d => d.id)
        .attr("fill", "black")
        .attr("stroke", "none")
        .attr("font-size", "0.8em");

      // Add zoom behavior
      const zoom = d3.zoom()
        .scaleExtent([0.1, 4]) // Set the zoom scale range
        .on("zoom", (event) => {
          zoomGroup.attr("transform", event.transform);
        });

      // Apply the zoom behavior to the SVG
      svg.call(zoom);

      // Update simulation at each tick
      simulation.on('tick', () => {
        links
          .attr('x1', d => d.source.x)
          .attr('y1', d => d.source.y)
          .attr('x2', d => d.target.x)
          .attr('y2', d => d.target.y);

        node
          .attr('transform', d => `translate(${d.x},${d.y})`);
      });

      // Function to handle drags event
      function drag(simulation) {
        function dragstarted(event) {
          if (!event.active) simulation.alphaTarget(0.3).restart();
          event.subject.fx = event.subject.x;
          event.subject.fy = event.subject.y;
        }

        function dragged(event) {
          event.subject.fx = event.x;
          event.subject.fy = event.y;
        }

        function dragended(event) {
          if (!event.active) simulation.alphaTarget(0);
          event.subject.fx = null;
          event.subject.fy = null;
        }

        return d3.drag()
          .on("start", dragstarted)
          .on("drag", dragged)
          .on("end", dragended);
      }
    }

    function dataFetch({source}) {
        console.log('dataFetch', {source});
        if(timeoutId !== undefined){
          clearTimeout(timeoutId);
        }
        fetch('http://localhost:8000/data')
            .then(response => response.json())
            .then(data => {
                draw({source, data});
                timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
            })
            .catch(error => {
                console.error('Error:', error);
                timeoutId = setTimeout(() => dataFetch({source: 'setTimeout'}), 5000);
            });
    }

    document.getElementById('refreshButton').addEventListener(
      'click',
      () => dataFetch({source: 'refreshButton'})
    );

    dataFetch({source: 'initialization'});

</details>

Voici le résultat du script pour le graphique 2D avec D3.js :
{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/d3-2d-force-graph-result.png?raw=true", alt="Resultat script graphique 2D D3.js", no_hover=true) }}

### 2. Graphique 3D avec three.js

Pour three js, il y besoin de beaucoup plus de lien pour importer tout ce que nous avons besoin. C'est pour ça que nous avons le code directement dans un fichier html.

Nous avons donc l'en tête d'un fichier html avec la balise `DOCTYPE`, `html` et `head`. Ensuite nous avons une balise style pour appliquer du css, notamment à la balise div avec l'id `visualization`, pour appliquer une hauteur de 500px. Je me suis rendu compte du fait que la div était caché dans le Google Colab, donc j'ai trouvé cette solution avec le css pour la forcer à prendre une certaine taille.

Nous avons également deux balises `script` pour importer `three` et `3d-force-graph`.

```html
<!DOCTYPE html>
<html>
<head>
    <style>
        #visualization {
            height: 500px;
        }
        body {
            margin: 0;
        }
        .node-label {
            font-size: 12px;
            padding: 1px 4px;
            border-radius: 4px;
            background-color: rgba(0,0,0,0.5);
            user-select: none;
        }
    </style>

    <script src="//unpkg.com/three"></script>
    <script src="//unpkg.com/3d-force-graph"></script>
</head>
```

Dans le body nous retrouvons l'intégralité du code y compris les balises utilisés pour afficher le graphique comme la div et le bouton pour refresh le graphique.

```html
<body>
    <div id="visualization"></div>
    <button id="refreshButton">Refresh</button>
```

Nous avons encore deux balises `script`, une pour importer le build de three.js et l'autre qui contient tout le code pour générer le graphique.

```html
    <script type="importmap">{ "imports": { "three": "https://unpkg.com/three/build/three.module.js" }}</script>
    <script type="module">
```

Nous retrouvons la même logique que le graphique 2D fait avec D3.js avec la fonction `threeDataFetch` pour récupérer les données et initialiser la variable `timeoutId`.

```js
    function threeDataFetch({source}) {
        console.log('threeDataFetch', {source});
        if(timeoutId !== undefined){
            clearTimeout(timeoutId);
        }
        fetch('http://localhost:8000/data')
        .then(response => response.json())
        .then(data => {
            draw({source, data});
            timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
        })
        .catch(error => {
            console.error('Error:', error);
            timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
        });
    }
```

Nous avons de nouveau, la fonction `draw` pour dessiner le graphique. Cette fonction est différente puisque nous n'utilisons plus D3.js mais three.js donc les méthodes et le fonctionnement est différent.

Grâce au commentaire, nous comprenons quelle partie agit sur quelle aspect du graphique.

- `.graphData`, de son nom, nous comprenons qu'elle sert à prendre la data à afficher.
- Le commentaire `Styling links` nous fait comprendre que les deux méthodes à sa suite sont pour styliser les liens entre les noeuds.
- Le commentaire `Styling nodes` nous fait comprendre que les méthodes à sa suite sont pour styliser les noeuds.

```js
    function draw({ source, data }) {
        const container = document.getElementById("visualization");

        let hoveredNode;

        const Graph = ForceGraph3D({
            extraRenderers: [new CSS2DRenderer()]
        })(container)
        .graphData(data)

        // Styling links
        .linkWidth(1)
        .linkColor("grey")

        // Styling nodes
        .nodeRelSize(3)
        .nodeOpacity(1)
        .nodeColor(node => {
            if (node == hoveredNode) {
                return "yellow";
            }
            return "grey";
        })
        .onNodeHover((node) => {
            hoveredNode = node;
            Graph.nodeColor(Graph.nodeColor())
        })
        .nodeThreeObject(node => {
            const nodeEl = document.createElement('div');
            nodeEl.textContent = node.id;
            nodeEl.style.color = "white";
            nodeEl.className = 'node-label';
            return new CSS2DObject(nodeEl);
        })
        .nodeThreeObjectExtend(true);
    }
```

<details>
    <summary>Voir le code en entier</summary>

    <!DOCTYPE html>
    <html>
    <head>
        <style>
          #visualization {
            height: 500px;
          }
          body {
              margin: 0;
          }
          .node-label {
            font-size: 12px;
            padding: 1px 4px;
            border-radius: 4px;
            background-color: rgba(0,0,0,0.5);
            user-select: none;
          }
        </style>

        <script src="//unpkg.com/three"></script>
        <script src="//unpkg.com/3d-force-graph"></script>
        <!--<script src="../../dist/3d-force-graph.js"></script>-->
    </head>
    <body>
      <div id="visualization"></div>
      <button id="refreshButton">Refresh</button>

      <script type="importmap">{ "imports": { "three": "https://unpkg.com/three/build/three.module.js" }}</script>
      <script type="module">
        import { CSS2DRenderer, CSS2DObject } from '//unpkg.com/three/examples/jsm/renderers/CSS2DRenderer.js';

        let timeoutId = undefined;

        function draw({ source, data }) {
            const container = document.getElementById("visualization");

            let hoveredNode;

            const Graph = ForceGraph3D({
              extraRenderers: [new CSS2DRenderer()]
            })(container)
            .graphData(data)

            // Styling links
            .linkWidth(1)
            .linkColor("grey")

            // Styling nodes
            .nodeRelSize(3)
            .nodeOpacity(1)
            .nodeColor(node => {
              if (node == hoveredNode) {
                return "yellow";
              }
              return "grey";
            })
            .onNodeHover((node) => {
              hoveredNode = node;
              Graph.nodeColor(Graph.nodeColor())
            })
            .nodeThreeObject(node => {
              const nodeEl = document.createElement('div');
              nodeEl.textContent = node.id;
              nodeEl.style.color = "white";
              nodeEl.className = 'node-label';
              return new CSS2DObject(nodeEl);
            })
            .nodeThreeObjectExtend(true);
        }

        function threeDataFetch({source}) {
          console.log('threeDataFetch', {source});
          if(timeoutId !== undefined){
            clearTimeout(timeoutId);
          }
          fetch('http://localhost:8000/data')
            .then(response => response.json())
            .then(data => {
                draw({source, data});
                timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
            })
            .catch(error => {
                console.error('Error:', error);
                timeoutId = setTimeout(() => threeDataFetch({source: 'setTimeout'}), 5000);
            });
        }

        document.getElementById('refreshButton').addEventListener(
          'click',
          () => threeDataFetch({source: 'refreshButton'})
        );

        threeDataFetch({source: 'initialization'});
      </script>
    </body>
    </html>
</details>

Voici le résultat de ce graphique :
{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/three-3d-force-graph-result.png?raw=true", alt="Resultat script graphique 3D three.js", no_hover=true) }}

La node jaune que nous apercevons n'est pas grise car je l'ai volontairement survoler avec la souris pour montrer l'effet de changement de couleur lors du hover d'un node.

## Conclusion sur cette expérience

J'ai décidé de parler de ces trois tâches, qui m'ont été attribué lors de ce stage au sein de Bashroom, parce que ceux sont les tâches qui m'ont appris le plus de chose et qui ont surtout pris la majorité du temps de mon stage. J'ai eu d'autres tâches durant mon stage, qui ont été très courtes et moins impactantes comme la traduction de code Python du Google Colab en R dans un Kaggle.

Ce stage a été pour moi une deuxième expérience avec Bashroom, j'ai intégré davantage l'entreprise en obtenant accès aux repos de l'entreprise avec mon compte personnel Bashroom pour y déposer le code que j'ai écrit durant ce stage. Ce stage a été bénéfique pour mettre en pratique de ce que j'ai appris à Ynov mais de mettre en pratique des compétences qui sont importantes au sein d'une entreprise comme ma communication, mon autonomie et mon sens des responsabilités.
