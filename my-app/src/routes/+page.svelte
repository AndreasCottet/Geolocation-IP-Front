<script>
    import {Button, ButtonGroup, GradientButton, Input, InputAddon, Label} from "flowbite-svelte";
    import {SearchOutline, UserCircleSolid} from "flowbite-svelte-icons";
    import axios from "axios";

    let address = '';
    let informations = {};

    async function handleClick() {
        // TODO : Regex pour controler
        try {
            const response = await axios.get('http://localhost:8080/address?address=' + address)
            informations = response.data;
            console.log(informations);
        } catch (error) {
            console.error(error);
        }
    }
</script>

<div class="mx-auto max-w-4xl text-center mt-24">
    <h1 class="text-4xl font-semibold mb-8">Entrer une adresse une IP</h1>
    <div class="mb-6 ">
        <ButtonGroup class="w-full">
            <InputAddon>
                <SearchOutline class="w-4 h-4 text-gray-500 dark:text-gray-400" />
            </InputAddon>
            <Input placeholder="Ex: 10.152.17.15" bind:value={address} />
        </ButtonGroup>
        <div class="flex flex-row gap-4 mx-auto w-fit">
            <Button href="/address/{address}" class="mt-8 py-4">Rechercher</Button>
            <GradientButton href="/adresses" class="mt-8 py-4" color="pinkToOrange">Voir les adresses déja enregistrées</GradientButton>
        </div>
    </div>
</div>
