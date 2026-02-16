# Test md

<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<title>Toulouse No Fibre</title>

<script src="https://cdn.jsdelivr.net/npm/ol@v10.7.0/dist/ol.js"></script>
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/ol@v10.7.0/ol.css">

<style>
body {
  margin:0;
  font-family: Arial, sans-serif;
  display:flex;
  justify-content:center;
}

.container {
  width:80%;
  max-width:1000px;
}

h1 {
  margin-top:20px;
}

p {
  color: white;
  background-color: rgba(19, 19, 19, 0.452);
  padding: 5px;
  border: 1px solid black;
}

#map {
  width:100%;
  height:60vh;
  min-height:400px;
  margin-top:10px;
}

.panel {
  margin-top:10px;
}
</style>
</head>

<body>

<div class="container">

  <h1>Carte des adresses non reliées à la fibre - Février 2026</h1>
  <div id="map"></div>
  
  <div class="panel">
  <p>
    <label>Choisir une commune :</label>
    <select id="communeSelect"></select>
    <br><br>
    <strong>IMB visibles dans la vue :</strong>
    <span id="count">0</span>
  </p>
  </div>  


  
  <p><b>Les chiffres : </b><br>
      La métropole compte 4502 adresses (imb) non reliées, la ville de Toulouse en compte 4389.<br>
      - La Métropole a 171 788 IMB, soit 99 016 IMB hors Toulouse, 
      dont 113 IMB non reliés à la fibre, ce qui représente 99.9 % des IMB reliés.<br>
      - <b>La Ville de Toulouse</b> posséde 72 772 IMB sur la commune.<br>
          - 94 % des IMB sont connectés. <br>
          - <b>6 % ne le sont toujours pas</b><br>
      (dont 2498 logements individuels, 1667 immeubles comptant entre 2 
      et 11 appartements et 224 immeubles de plus de 12 appartements).
  </p>
  </div>


  <div>
  <p><font size="1">
    Visualisation des IMB non reliées à la fibre. Les données sont issues
    de <a href=https://data.arcep.fr/fixe/maconnexioninternet/base_imb/2025_T3/departement/index.html> Open Data Arcep chiffres du troisième tirmestre 2025</a>.
    <br>
    Données récupérées au format csv + sélection des IMB "en cours de deploiement" (champ etat_imb).
    <br>
    Communes de Toulouse Métropole issues de OpenData Toulouse 
    <a href=https://data.toulouse-metropole.fr/pages/home/>https://data.toulouse-metropole.fr/pages/home/</a>.
    <br>
    Utilisation de QGIS <a href="https://qgis.org"><img src="https://qgis.org/styleguide/visual/qgis-icon-black32.svg"></a>  pour sélectionner les imb non reliées sur Toulouse Métropole.
  </p>
  </div>

<iframe src="https://framaforms.org/toulousenofibre-1771272934" width="100%" height="800" border="0"></iframe>

</div>

<script>

/* ======================
   Carte
====================== */
const map = new ol.Map({
  target: 'map',
  layers: [
    new ol.layer.Tile({
      source: new ol.source.OSM()
    })
  ],
  view: new ol.View({
    center: ol.proj.fromLonLat([1.41,43.64]),
    zoom: 11
  })
});

/* ======================
   Communes
====================== */
const communeSource = new ol.source.Vector({
  url: "./data/communes.geojson",
  format: new ol.format.GeoJSON()
});

const communeLayer = new ol.layer.Vector({
  source: communeSource,
  style: new ol.style.Style({
    stroke: new ol.style.Stroke({color:"black", width:2}),
    fill: new ol.style.Fill({color:"rgba(243,243,243,0.1)"})
  })
});
map.addLayer(communeLayer);

/* ======================
   IMB source
====================== */
const imbSource = new ol.source.Vector({
  url: "./data/imb_t3_2025_edd.geojson",
  format: new ol.format.GeoJSON()
});

/* ======================
   Cluster
====================== */
const clusterSource = new ol.source.Cluster({
  distance: 40,
  source: imbSource
});

const clusterLayer = new ol.layer.Vector({
  source: clusterSource,
  style: function(feature){
    const size = feature.get('features').length;

    if(size === 1){
      return new ol.style.Style({
        image: new ol.style.Circle({
          radius:6,
          fill:new ol.style.Fill({color:"red"}),
          stroke:new ol.style.Stroke({color:"#fff",width:1})
        })
      });
    }

    return new ol.style.Style({
      image: new ol.style.Circle({
        radius:12,
        fill:new ol.style.Fill({color:"orange"})
      }),
      text:new ol.style.Text({
        text:size.toString(),
        fill:new ol.style.Fill({color:"#fff"})
      })
    });
  }
});
map.addLayer(clusterLayer);

/* ======================
   Menu déroulant
====================== */
const select = document.getElementById("communeSelect");

communeSource.once("change", ()=>{
  if(communeSource.getState() === "ready"){
    communeSource.getFeatures()
      .sort((a,b)=>a.get("libelle").localeCompare(b.get("libelle")))
      .forEach(f=>{
        const opt = document.createElement("option");
        opt.textContent = f.get("libelle");
        select.appendChild(opt);
      });
  }
});

select.addEventListener("change", ()=>{
  const libelle = select.value;

  const commune = communeSource.getFeatures()
    .find(f => f.get("libelle") === libelle);

  if(!commune) return;

  map.getView().fit(commune.getGeometry(), {
    padding:[30,30,30,30]
  });
});

/* ======================
   Compteur fiable
====================== */
function updateCount(){

  const extent = map.getView().calculateExtent(map.getSize());

  let count = 0;

  imbSource.forEachFeature(feature => {
    const coord = feature.getGeometry().getCoordinates();
    if(ol.extent.containsCoordinate(extent, coord)){
      count++;
    }
  });

  document.getElementById("count").textContent = count;
}

/* attendre chargement IMB */
imbSource.once("change", ()=>{
  if(imbSource.getState() === "ready"){
    updateCount();
  }
});

/* mise à jour au zoom / déplacement */
map.on("moveend", updateCount);

</script>

</body>
</html>
