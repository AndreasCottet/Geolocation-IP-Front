<script>
    import {onMount} from 'svelte';
    import {Button, Checkbox, Fileupload, Helper, Input, Label, Modal, Spinner} from "flowbite-svelte";
    import {
        CloseOutline,
        CloudArrowUpOutline,
        FileCirclePlusOutline,
        SearchOutline,
        TrashBinOutline
    } from "flowbite-svelte-icons";
    import axios from "axios";
    import * as XLSX from "xlsx";
    import Swal from "sweetalert2";

    let addressIP = '';
    let map;
    let valueDisplayAllAddress = false;

    let currentMarker = null;

    let allMarkersAddress = []
    let allAddressesIP = []
    let allAddressesIPQuery = []
    $: allAddressesIPQuery = allAddressesIP.map(addressIP => addressIP.query);

    let loading = true;
    let firstPart = true;

    let currentInformations = null;

    let defaultModal = false;
    let listSearchAddress = []

    let monitorAddress = []

    onMount(async () => {
        await import('leaflet/dist/leaflet.css');
        await import('leaflet');

        loadMap();

        const resAllAddresses = await axios.get("http://localhost:8080/address")
        allAddressesIP = resAllAddresses.data
    });

    function displayAllAddress(display) {
        if (display) {
            allAddressesIP.forEach(address => {
                let marker = createMarker(address)
                marker.addTo(map).openPopup()
                allMarkersAddress.push(marker)
            })
        } else {
            allMarkersAddress.forEach(marker => marker.remove())
            allMarkersAddress = []
        }
        valueDisplayAllAddress = display
    }

    function loadMap() {
        map = L.map('map').setView([44.85, -0.54], 14);
        L.tileLayer('https://{s}.basemaps.cartocdn.com/rastertiles/voyager/{z}/{x}/{y}{r}.png', {
            attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors &copy; <a href="https://carto.com/attributions">CARTO</a>',
            maxZoom: 20
        }).addTo(map);
    }

    function createMarker(addressInformations) {
        let div = L.DomUtil.create('div')
        div.innerHTML = `<p>Adresse IP : ${addressInformations?.query}.<br>Latitude : ${addressInformations?.lat}, Longitude : ${addressInformations?.lon}.</p>`
        let buttonMarker = L.DomUtil.create('button', '', div)
        buttonMarker.className = 'w-full text-white bg-primary-700 hover:bg-primary-800 focus:ring-4 focus:ring-primary-300 font-medium rounded-lg text-sm px-5 py-2.5 me-2 mb-2 dark:bg-primary-600 dark:hover:bg-primary-700 focus:outline-none dark:focus:ring-primary-800'
        buttonMarker.innerHTML = 'Voir plus d\'informations'
        buttonMarker.informations = addressInformations;

        buttonMarker.onclick = (e) => {
            currentInformations = e.explicitOriginalTarget?.informations
        }

        return L.marker([addressInformations?.lat, addressInformations?.lon], {
            icon:
                new L.Icon({iconUrl: 'https://unpkg.com/leaflet@1.6.0/dist/images/marker-icon.png'})
        }).bindPopup(div)
    }

    function addMarker(addressInformations) {
        let marker = createMarker(addressInformations)

        marker.addTo(map).openPopup()
        map.setView([addressInformations.lat, addressInformations.lon], 14)

        if (currentMarker) {
            currentMarker.remove()
        }
        currentMarker = marker
    }

    async function saveAllAddressesIP(addressInformations) {
        const resAllAddresses = await axios.get("http://localhost:8080/address")
        allAddressesIP = resAllAddresses.data
    }

    async function searchAddressIP() {
        loading = true
        try {
            let resIP = await axios.get('http://localhost:8080/address/' + addressIP)
            if (resIP.data.length === 0) {
                loading = false

                const result = await Swal.fire({
                    title: "Aucune adresse IP trouvée",
                    text: "Voulez-vous l'ajouter au système ?",
                    icon: "warning",
                    showCancelButton: true,
                    confirmButtonColor: "#3085d6",
                    cancelButtonColor: "#d33",
                    confirmButtonText: "Oui !"
                });

                if (!result.isConfirmed) {
                    return
                }
                loading = true

                resIP = await axios.post('http://localhost:8080/address', {addresses: [addressIP]})
                await saveAllAddressesIP()

                if (resIP.data.length > 0) {
                    addMarker(resIP.data[0])
                }
            } else {
                addMarker(resIP.data[0])
            }
        } catch (e) {
            console.error(e)
            Swal.fire("Erreur", "Une erreur est survenue lors de la recherche de l'adresse IP.<br> Message : " + (e.response?.data ?  e.response.data : "non défini"), "error")
        }
        loading = false
    }

    async function exportInformations() {
        try {
            const res = await axios.get('http://localhost:8080/address/export')
            console.log(res)
        } catch (e) {
            console.error(e)
        }
    }

    function handleChangeFile (event) {
        const file = event.target.files[0];
        parseXLS(file);
    }

    function parseXLS (file) {
        let reader = new FileReader();

        reader.onload = function(e) {
            let data = e.target.result;
            let workbook = XLSX.read(data, {
                type: 'binary'
            });


            workbook.SheetNames.forEach(function(sheetName) {
                let XL_row_object = XLSX.utils.sheet_to_row_object_array(workbook.Sheets[sheetName]);
                XL_row_object.forEach(address => {
                    if (address.address && isNotIncludedStr(address.address, listSearchAddress)) {
                        listSearchAddress.push(address.address)
                        listSearchAddress = listSearchAddress
                    }
                })
            })

        };

        reader.onerror = function(ex) {
            console.log(ex);
        };

        reader.readAsBinaryString(file);
    }



    function isNotIncludedStr(str, list) {
        return !list.some(item => item.includes(str));
    }

    async function updateAdresseIP(addressIP) {
        loading = true
        try {
            const res = await axios.put('http://localhost:8080/address/', {address: addressIP})
            currentInformations = null
            addMarker(res.data)
            await saveAllAddressesIP()
            Swal.fire("Adresse IP mise à jour", "L'adresse IP a bien été mise à jour.", "success")
        } catch (e) {
            console.error(e)
            Swal.fire("Erreur", "Une erreur est survenue lors de la mise à jour de l'adresse IP.<br/>Message : " + (e.response?.data ?  e.response.data : "non défini"), "error")
        }
        loading = false
    }

    async function deleteAdresseIP(addressIP) {
        loading = true
        try {
            await axios.delete('http://localhost:8080/address/' + addressIP)

            if (currentMarker) {
                currentMarker.remove()
                currentMarker = null
            }
            currentInformations = null

            await saveAllAddressesIP()

            const tmpValueDisplayAllAddress = valueDisplayAllAddress
            displayAllAddress(false)
            displayAllAddress(tmpValueDisplayAllAddress)
            Swal.fire("Adresse IP supprimée", "L'adresse IP a bien été supprimée.", "success")
        } catch (e) {
            console.error(e)
            Swal.fire("Erreur", "Une erreur est survenue lors de la suppression de l'adresse IP.<br>Message : " + (e.response?.data ?  e.response.data : "non défini"), "error")
        }
        loading = false
    }

    function handleShowAllAddress(event) {
        displayAllAddress(event.target.checked)
    }

    async function handleAddLotAddress() {
        listSearchAddress = listSearchAddress.filter(address => address !== '')
        if (listSearchAddress.length === 0) {
            return
        }

        defaultModal = false
        loading = true

        try {
            const res = await axios.post('http://localhost:8080/address', {addresses: listSearchAddress})
            console.log(res.data)
            await saveAllAddressesIP()
            res.data.forEach((address) => {
                addMarker(address)
            })
            Swal.fire("Adresses IP ajoutées", "Les adresses IP ont bien été ajoutées.", "success")
        } catch (e) {
            console.error(e)
            Swal.fire("Erreur", "Une erreur est survenue de l'ajout des addresses IP.<br> Message : " + (e.response?.data ?  e.response.data : "non défini"), "error")
        }
        loading = false
    }

    function isAddress (address)  {
        const ipRegex = /^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)$/;
        return ipRegex.test(address)
    }

    function convertToCSV(data) {
        let csv = '';
        csv += Object.keys(data[0]).join(',') + '\n';
        data.forEach((row) => {
            csv += Object.values(row).join(',') + '\n';
        });
        return csv;
    }

    function handleCreateCSV() {
        // Données d'exemple (supposons que vous avez ces données dans votre application)
        // const exampleData = [
        //     { name: 'John', age: 30, city: 'New York' },
        //     { name: 'Alice', age: 25, city: 'Los Angeles' },
        //     { name: 'Bob', age: 35, city: 'Chicago' }
        // ];

        const csvData = convertToCSV(monitorAddress);
        const blob = new Blob([csvData], { type: 'text/csv' });

        const url = URL.createObjectURL(blob);

        const downloadLink = document.createElement('a');
        downloadLink.href = url;

        // Simuler un clic sur le lien pour déclencher le téléchargement
        downloadLink.click();

        // Nettoyer l'URL après le téléchargement
        URL.revokeObjectURL(url);
    }

    function handleMonitorIP(informations) {
        console.log(monitorAddress)
        if (monitorAddress.some(item => item.query === informations.query)) {
            return
        }
        monitorAddress.push(informations)
        monitorAddress = monitorAddress
    }
</script>

<div class="max-w-lg w-full fixed z-[99999] top-4 left-16">
    <Input id="search" placeholder="Ex : 172.10.24.5" size="lg" bind:value={addressIP}>
        <SearchOutline slot="left" class="w-6 h-6 text-gray-500" />
        <Button slot="right" size="sm" type="submit" on:click={searchAddressIP}>Voir ses informations</Button>
    </Input>
    {#if addressIP && !isNotIncludedStr(addressIP, allAddressesIPQuery) }
        <div class="mt-1 rounded-xl border border-gray-200 bg-white px-4 py-2 flex flex-col gap-2">
            {#each allAddressesIPQuery as query}
                {#if query.includes(addressIP)}
                    <div class="flex justify-between items-center">
                        <p>{query}</p>
                        <Button on:click={() => { addressIP = query; searchAddressIP(); }} color="blue">Voir sur la carte</Button>
                    </div>
                {/if}
            {/each}
        </div>
    {/if}
    <Button class="z-[99999] mt-2 flex gap-2" pill on:click={() => (defaultModal = true)}><FileCirclePlusOutline/> Importer des adresses depuis un fichier.</Button>
</div>


<div class="bg-white border border-gray-200 fixed z-[9999] top-4 right-4 px-3 py-2 rounded-lg">
    <Checkbox on:change={handleShowAllAddress}>Afficher toutes les adresses IP collectées</Checkbox>
</div>

{#if currentInformations}
    <div class="bg-white border border-gray-200 fixed z-[9999999] left-4 bottom-6 py-6 px-8 rounded-lg ">
        <h1 class="font-semibold mb-4">Récapitulatif des informations</h1>
        <button on:click={() => currentInformations = null} class="absolute top-2 right-2 hover:text-red-800 text-red-600">
            <CloseOutline size="lg"/>
        </button>
        <div class="flex gap-8 mb-6">
            <div class="font-semibold space-y-2">
                <p>Adresse IP :</p>
                <p>Pays : </p>
                <p>Continent : </p>
                <p>Ville : </p>
                <p>Latitude :</p>
                <p>Longitude :</p>
                <p>Organisation :</p>
                <p>ISP :</p>
                <p>AS :</p>
                <p>AS NAME :</p>
                <p>Region :</p>
                <p>Code postal :</p>
            </div>
            <div class="space-y-2">
                <p>{currentInformations.query}</p>
                <p>{currentInformations.country}</p>
                <p>{currentInformations.continent}</p>
                <p>{currentInformations.city}</p>
                <p>{currentInformations.lat}</p>
                <p>{currentInformations.lon}</p>
                <p>{currentInformations.org}</p>
                <p>{currentInformations.isp}</p>
                <p>{currentInformations.as}</p>
                <p>{currentInformations.asname}</p>
                <p>{currentInformations.regionName}</p>
                <p>{currentInformations.zip}</p>
            </div>
        </div>
        <Button on:click={() => deleteAdresseIP(currentInformations.query)} color="red"><TrashBinOutline/></Button>
        <Button on:click={() => updateAdresseIP(currentInformations.query)} color="blue"><CloudArrowUpOutline/> Mettre à jour l'addresse IP</Button>
        <Button on:click={() => handleMonitorIP(currentInformations)} color="blue"><CloudArrowUpOutline/> Surveiller l'adresse IP</Button>
    </div>
{/if}

<Modal title="Importer des adresses depuis un fichier" bind:open={defaultModal} dialogClass="fixed top-0 start-0 end-0 h-modal md:inset-0 md:h-full w-full p-4 flex z-[9999999]" backdropClass="fixed inset-0 z-[999999] bg-gray-900 bg-opacity-50 dark:bg-opacity-80">
    <h1 class="font-semibold">Les adresses IP :</h1>
    <div class="max-h-96 overflow-y-scroll flex flex-col gap-2 border-b border-gray-200">
        {#each listSearchAddress as _, i}
               <div class="flex gap-2">
                   <Input type="text" bind:value={listSearchAddress[i]}/>
                   <Button on:click={() => { listSearchAddress.splice(i, 1); listSearchAddress = listSearchAddress}} class="flex items-center"><CloseOutline/></Button>
               </div>
        {/each}
        <Button on:click={() => { listSearchAddress.push(''); listSearchAddress = listSearchAddress}} >Ajouter une adresse IP</Button>
    </div>
    <Label for="with_helper" class="pb-2">Upload file</Label>
    <Fileupload id="with_helper" class="mb-2" on:change={handleChangeFile} />
    <Helper>XSL (MAX. 3Mo).</Helper>
    <svelte:fragment slot="footer">
        <Button on:click={handleAddLotAddress}>Ajouter</Button>
        <Button color="alternative" on:click={() => defaultModal = false}>Annuler</Button>
    </svelte:fragment>
</Modal>

{#if loading}
    <div class="fixed z-[9999999] bg-primary-800/50 w-full h-full flex justify-center items-center">
        <div class="bg-white flex justify-center px-8 pt-4 pb-8 rounded-xl border border-gray-200 flex-col items-center">
            <h1 class="text-2xl font-semibold mb-4">Chargement en cours...</h1>
            <Spinner />
        </div>
    </div>
{/if}

{#if firstPart}
    <div class="fixed z-[9999999] bg-primary-800/50 w-full h-full flex justify-center items-center">
        <div class="bg-white flex justify-center px-8 pt-4 pb-8 rounded-xl border border-gray-200 flex-col">
            <h1 class="text-2xl font-semibold mb-4">Commencer par entrée une adresse IP</h1>
            <Input type="text" placeholder="Ex : 172.10.24.5" bind:value={addressIP}/>
            <Button on:click={() => { firstPart = false; searchAddressIP()}} class="mt-4" disabled={!isAddress(addressIP)}>Rechercher</Button>
        </div>
    </div>
{/if}

<div class="bg-white border border-gray-200 fixed z-[9999] bottom-6 right-4 rounded-lg w-64">
    <h1 class="p-2 bg-primary-600 text-white rounded-t-lg">Addresse à surveiller</h1>
    <div class="px-3 py-2">
        {#each monitorAddress as address}
            <div class="flex justify-between items-center py-2 border-b border-gray-200 hover:bg-primary-200">
                <Button on:click={() => addMarker(address)}>{address.query}</Button>
                <Button on:click={() => monitorAddress = monitorAddress.filter(item => item.query !== address.query)} color="red" class="w-fit"><TrashBinOutline/></Button>
            </div>
        {/each}
        {#if monitorAddress.length}
            <Button on:click={handleCreateCSV} class="w-full">Exporter</Button>
        {/if}
    </div>
</div>

<div id="map" class="w-full h-screen"></div>
