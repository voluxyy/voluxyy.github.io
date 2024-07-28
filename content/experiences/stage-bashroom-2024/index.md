+++
title = "Stage Bashroom 2024"
+++

# Stage Bashroom 2024

{% alert(important=true) %}
Voici une liste des tâches que j'ai réalisées durant mon stage.
{% end %}

- Apprendre à utiliser Godot pour exporter un projet Godot en version web via un script.
- Créer des graphiques avec D3.js pour afficher les futures sessions de formation de l'entreprise.
- Réaliser des graphiques 2D avec D3.js et des graphiques 3D avec three.js pour la formation en data science.
- Convertir des graphiques réalisés pour la formation en data science en langage R.
- Convertir un projet Google Colab (Python) en projet Kaggle (R).
- Créer un exercice pour une formation en utilisant Electron et C++.

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

En observant les arguments, on retrouve <mark>preset</mark> et <mark>path</mark>. <strong>Path</strong> qui signifie tout simplement le chemin absolu ou relatif vers le projet godot, alors que <strong>Preset</strong> est un peu complexe à comprendre.

J'ai compris qu'ajouter un present d'exportation au projet comme sur l'image suivante :

{{ image(url="https://github.com/voluxyy/voluxyy.github.io/blob/main/static/experiences/stage-bashroom-2024/godot-export-menu.png?raw=true", alt="Godot adding export option menu", no_hover=true) }}

Permet de générer, si non existant, le fichier `export_presets.cfg` à la racine du projet Godot et ajoute la configuration du preset dedans avec les options qu'on lui affecte.

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

## 2. Graphique D2.js pour afficher les futures formations

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

### 3. Création du graphique

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

