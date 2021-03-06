<template>
    <section>
        <figure class="background" :class="{'full':identified}"></figure>



        <figure class="power-out" @click="logout" :class="{'show':identified}">
            <i class="fa fa-power-off"></i>
        </figure>
        <figure class="doodad" :class="{'gone':!identified}">
            <img src="assets/doodad.png"/>
        </figure>
        <section class="error" :class="{'show':error}">{{error}}</section>
        <figure class="logo top" :class="{'gone':!identified}">Scattellet</figure>
        <section class="container">
            <figure class="logo" :class="{'gone':identified}">Scattellet</figure>
            <section class="network" :class="{'hidden':identified}">
                <section class="input-container">
                    <!--<label>Chain</label>-->
                    <!--<input placeholder="Domain or IP" v-model="selectedNetworkString" />-->
                </section>
            </section>
            <section style="position:relative;">
                <figure class="token-account" :class="{'gone':!identified}" @click="changingTokenAddress = !changingTokenAddress">
                    <i class="fa fa-at" v-if="!changingTokenAddress"></i>
                    <i class="fa fa-check" v-else></i>
                </figure>
                <section class="input-containers" :class="{'open':identified}">
                    <section class="input-container">
                        <input placeholder="Recipient" v-model="recipient" />
                    </section>
                    <section class="input-container" v-if="!changingTokenAddress">
                        <input placeholder="SYM" v-model="symbol" class="symbol" />
                        <input placeholder="Amount" class="amount" v-model="amount" />
                        <input placeholder="Memo" class="memo" v-model="memo" />
                    </section>
                    <section class="input-container" v-else>
                        <input placeholder="Token Address" class="token-address" v-model="tokenAddress" />
                    </section>
                </section>
                <button class="animated" :disabled="transferring" @click="identifyOrSend" :class="{'send':identified, 'tada':transferred}">{{identified ? 'SEND' : 'ID'}}</button>
            </section>
        </section>
        <section class="trx" :class="{'show':this.identified && trx.length}">Last Transaction ID: {{trx}}</section>
        <section :class="{'show':this.identified && this.symbol.length}" class="balance">You have <b>[ {{balance}} {{symbol}} ]</b> tokens</section>
    </section>
</template>

<script>
    import { mapActions, mapGetters, mapState } from 'vuex'
    import * as Actions from '../store/constants';
    import {RouteNames} from '../vue/Routing'

    import Eos from 'eosjs';

    let networkTimeout = null;

    const network = {
        blockchain:'eos',
        host:'api.travelchain.io',
        port:location.protocol === 'https:' ? 443 : 80,
        chainId:'0443d062bb782b32bdc07a65273e1696c9a28749c047124927c7160897cacd28'
    };

    export default {
        data(){ return {
            identified:false,
            recipient:'',
            symbol:'TT',
            amount:'',
            memo:'',
            balance:'0',
            eos:null,
            transferring:false,
            transferred:false,
            trx:'',
            error:null,
            selectingNetwork:false,
            selectedNetworkString:'',
            changingTokenAddress:false,
            tokenAddress:'eosio.token'
        }},

        computed: {
            ...mapState([
                'scatter',
                'scateos',
                'selectedNetwork'
            ]),
            ...mapGetters([
                'identity',
                'account'
            ])
        },
        mounted(){
            setInterval(() => this.getBalance(), 5000);
        },
        methods: {
            async identifyOrSend(){
                if(!this.identity) {
                    const requiredFields = {accounts: [network]};
                    await this.scatter.getIdentity(requiredFields);
                } else this.transfer();
            },
            toggleSelectingNetwork(){
                this.selectingNetwork = !this.selectingNetwork;
            },
            async logout(){
                await this.scatter.forgetIdentity();
            },
            async transfer(){
                this.error = null;
                if(this.transferring) return;
                if(!this.recipient.length) return this.error = 'No Recipient';
                if(!this.amount.length || parseFloat(this.amount) <= 0) return this.error = 'Amount must be greater than 0';
                this.transferring = true;
                this.transferred = false;
                const scateos = this.scatter.eos(network, Eos, {chainId:network.chainId}, location.protocol.replace(':', ''));
                const contract = await scateos.contract(this.tokenAddress);
                const transferred = await contract.transfer(this.account.name, this.recipient, `${this.amount} ${this.symbol}`, this.memo).catch(error => {
                    if(typeof error === 'object') this.error = error.message;
                    else this.error = JSON.parse(error).error.details[0].message.replace('condition: assertion failed: ', '');
                    if(this.error.trim() === 'unknown key:') this.error = 'No such account';
                    return null;
                });
                this.transferring = false;
                if(transferred) {
                    this.transferred = true;
                    this.trx = transferred.transaction_id;
                    this.getBalance();
                    this.amount = '';
                }

                setTimeout(() => {
                    this.transferred = false;
                }, 1000);
            },
            test(){
                this.identified = !this.identified;
            },
            async getBalance(){
                if(!this.account || !this.symbol.length) {
                    this.balance = '0';
                    return false;
                }
                const balances = await Eos({httpEndpoint:`${location.protocol}//${network.host}`, chainId:network.chainId}).getTableRows({
                    json:true,
                    code:this.tokenAddress,
                    scope:this.account.name,
                    table:'accounts',
                    limit:500
                }).then(res => res.rows).catch(() => []);

                const row = balances.find(row => row.balance.split(" ")[1].toLowerCase() === this.symbol.toLowerCase());
                this.balance = row ? row.balance.split(" ")[0] : 0;
            },
            ...mapActions([
                Actions.SET_SCATEOS,
                Actions.SET_NETWORK,
            ])
        },
        watch:{
            identity(){
                if(this.identity) setTimeout(() => {
                    this.identified = true;
                    this.getBalance();
                }, 200);
                else this.identified = false;
            },
            symbol(){
                this.getBalance();
            },
            tokenAddress(){
                this.getBalance();
            },
        }

    }

</script>

<style lang="scss">
    .screen {
        position: relative;
        left:0;
        transition: all 0.5s ease;
        transition-property: left;

        &.out-right {
            left:100%;
        }
    }

    .network {
        position: relative;
        z-index:2;
        opacity:1;
        visibility: visible;
        transition: all 0.2s ease;
        transition-property: opacity, visibility;

        &.hidden {
            opacity:0;
            visibility: hidden;
        }

        .input-container {
            overflow: hidden;
            margin-bottom:20px;

            label {
                border-top-left-radius: 50px;
                border-bottom-left-radius: 50px;
                width:60px;
                text-align:center;
                height:28px;
                line-height:29px;
                font-size:11px;
                float:left;
                text-transform: uppercase;
                font-weight:800;
                background:rgba(255,255,255,0.3);
                color:#fff;
                text-shadow: 0 1px 2px rgba(0,0,0,0.2);
            }

            input {
                padding:0 10px;
                height:28px;
                line-height:26px;
                font-size:13px;
                float:left;
                border-top-left-radius: 0;
                border-bottom-left-radius: 0;
                width:250px;
                margin-bottom:10px;
                font-family: 'Open Sans', sans-serif;
                background:rgba(255,255,255,0.2);
                color:#fff;
                text-shadow: 0 1px 2px rgba(0,0,0,0.3);
            }
        }
    }
</style>