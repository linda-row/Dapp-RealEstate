<template>
    <!-- ======= Header ======= -->
    <header id="header" class="fixed-top d-flex align-items-center">
        <div class="container d-flex align-items-center justify-content-between">

            <div class="logo">
                <!-- Uncomment below if you prefer to use an image logo -->
                <a href="/">
                    <img src="/img/logo-with-name.png" alt="" class="img-fluid">
                    <!--<h1 style="display:inline;margin-left:10px">Hmonpa</h1>-->
                </a>
            </div>

            <nav id="navbar" class="navbar">
                <ul>
                    <li><a class="nav-link scrollto" href="/#hero">Home</a></li>
                    <li>
                        <NuxtLink class="nav-link scrollto" to="properties">Properties</NuxtLink>
                    </li>
                    <li>
                        <NuxtLink v-if="userLogged" class="getstarted scrollto" to="publish">Publish a new property</NuxtLink>
                    </li>
                    <li>
                        <!-- Modal -->
                        <a 
                            v-if="userLogged"
                            id="account"
                            class="access scrollto"
                            data-aos="fade-up" 
                            data-aos-delay="100"
                            data-bs-toggle="modal"
                            :data-bs-target="'#account_' + account"
                            style="cursor: pointer"
                        >
                            {{ account }}
                        </a>
                        <!-- End Modal -->
                        <div v-else>
                            <NuxtLink class="access scrollto-access" to="access" style="display:inline">Sign in</NuxtLink>
                            <NuxtLink class="access scrollto-access" to="create" style="display:inline">Sign up</NuxtLink>
                        </div>
                    </li>
                    <li>
                        <a v-if="!account" class="metamask" style="cursor: pointer" @click="connectWallet"></a>
                        <a v-if="account && !userLogged" id="account">{{ account }}</a>
                    </li>
                    <li>
                        <a v-if="userLogged" class="nav-link scrollto" style="cursor: pointer" @click="logout">Logout</a>
                    </li>
                </ul>
                <i class="bi bi-list mobile-nav-toggle"></i>
            </nav>
            <!-- .navbar -->

        </div>
        <!-- Modal -->
        <div
            v-if="userLogged"
            :id="'account_' + account"
            :class="fade"
        >
            <div
                class=" modal-dialog modal-lg modal-dialog-centered modal-dialog-scrollable"
            >
                <div class="modal-content">
                    <div class="modal-header">
                        <h6 style="text-align:center">{{ userLogged[0] }}</h6>
                        <button
                            type="button"
                            class="btn-close"
                            data-bs-dismiss="modal"
                            aria-label="Close"
                            @click="pause"
                        ></button>
                    </div>
                    <div class="modal-body" style="padding: 40px;text-align:center">
                        <!--<span class="line"></span>-->
                        <img id="ipfsImage" :src="`https://ipfs.io/ipfs/${userLogged[5]}`">
                        <h6 class="params">Name: <b>{{ userLogged[1] }}</b></h6>
                        <h6 class="params">ID card: <b>{{ userLogged[4] }}</b></h6>
                        <h6 class="params">Email: <b>{{ userLogged[2] }}</b></h6> 
                        <h6 class="params">User since: <b>{{ new Date(userLogged[3]*1000).toLocaleString() }}</b></h6>
                        <div style="margin-top: 30px">
                            <h6 class="params">My Properties: <b>{{ myProperties.length }}</b></h6>
                            <div 
                                v-if="myProperties.length > 0"
                                v-for="(prop, index) in myProperties"
                                :key="index"
                                style="margin-top: 10px"
                            >
                                <h6><b>{{ prop }}</b></h6>
                            </div>
                        </div>
                        <div style="margin-top: 30px" v-if="tokensRender">
                            <h6 class="params">My Tokens: <b>{{ totalUserTokens }}</b></h6>
                            <div 
                                v-if="myTokens.length > 0"
                                v-for="(token, index) in myTokens"
                                :key="index"
                            >
                                <h6>Token from property: <b>{{ token.idProperty }}</b></h6>
                                <h6>Number of tokens: <b>{{ token.tokens }}</b></h6>
                            </div>
                        </div>
                    </div>
                    <div class="modal-footer">
                    <button
                        type="button"
                        class="buy-tickets"
                        data-bs-dismiss="modal"
                        @click="pause()"
                    >
                        Close
                    </button>
                    </div>
                </div>
            </div>
        </div>
    </header>
    <!-- End Header -->
</template>

<script>
import { Web3Controller } from '@/src/controllers/web3';
import auth from '@/src/services/auth';
import swal from 'sweetalert';
import Swal from 'sweetalert2';

export default {
    computed: {
        userLogged() {
            let user = auth.getUserLogged();
            let data = "";
            if(user) data = user.split(",");

            return data;
        }
    },
    data() {
        const account = null;
        return {
            fade: "modal fade",
            autoplay: true,
            account,
            properties: [],
            tokenizedProperties: [],
            myProperties: [],
            myTokens: [],
            tokensRender: false,
            totalUserTokens: 0
        }
    },
    async beforeMount(){
        await Web3Controller.init();
        this.account = await Web3Controller.currentAccount();

        // Checking every second if MetaMask have an account
        var accountInterval = setInterval(async() => {
            this.account = await Web3Controller.currentAccount();
            // If userLogged and account changes, force the logout
            if((!this.account && this.userLogged) || (this.userLogged && this.isAccountChanged()))
            {
                auth.logoutUser();
                this.logout();
                window.location.href = "/";
            }
        }, 1000);

        await this.renderProperties();
        this.getMyProperties();
        await this.getMyTokens();
    },
    methods: {
        isAccountChanged(){
            return this.account.toLowerCase() != this.userLogged[0].toLowerCase();
        },

        async connectWallet() {
            this.account = await Web3Controller.currentAccount();            
            if (!this.account)
            {
                Swal.fire({
                    title: "Vinculation fails!",
                    text: "Connect your MetaMask wallet or review your extension",
                    icon: "error"
                })
            } else {
                Swal.fire({
                    title: "Great!",
                    text: 'Your account with public key ' + this.account + ' is already connected',
                    icon: 'success'
                });
            }
        },

        getMyProperties(){
            for (let i=0; i<this.properties.length; i++)
            {
                if (this.properties[i].owner.toLowerCase() == this.account){
                    this.myProperties.push(this.properties[i].id);
                }
            }
        },
        
        async logout() {
            Swal.fire({
                title: 'Are you sure you want to exit?',
                icon: 'warning',
                showDenyButton: true,
                confirmButtonColor: '#00F838',
                confirmButtonText: 'Yes, I want to go out',
                denyButtonText: 'No, I want to stay'
            }).then((res) => {
                if(res.isConfirmed) {
                    let user = auth.getUserLogged();
                    auth.logoutUser(user);
                    this.myProperties   = [];
                    this.myTokens       = [];
                    window.location.href = "/";
                }
            });
        },

        // Catch the current created properties
        async renderProperties(){
            try {
                this.properties             = [];
                this.tokenizedProperties    = [];

                const invalidAddr   = 0x0000000000000000000000000000000000000000;
                let numProperties   = await Web3Controller.Properties.cnt();

                for (let i = 0; i < numProperties.toNumber(); i++)
                {
                    let prop = await Web3Controller.Properties.properties(i);
                    let owner = prop.owner;

                    if (owner != invalidAddr)
                        this.properties.push(prop);
                }
                
            } catch (err) {
                console.log(err);
            }
        },

        async getMyTokens(){
            try {
                this.myTokens = [];
                this.myFilteredTokens = [];

                let numTokensPurchased   = await Web3Controller.Properties.cntTokens();
                for (let i = 0; i < numTokensPurchased.toNumber(); i++){
                    let tokenIndex = await Web3Controller.Properties.ownershipTokens(i);
                    
                    if(tokenIndex.owner.toLowerCase() == this.account)
                    {
                        this.myTokens.push(tokenIndex);
                    }
                }
                
                for (let i = 0; i < this.myTokens.length; i++)
                {
                    let firstToken = this.myTokens[i].id;
                    let j = i;
                    for (j = i+1; j < this.myTokens.length; j++)
                    {
                        if(this.myTokens[j].id == firstToken){
                            let tokensA;
                            let tokensB;
                            if(typeof(this.myTokens[i].tokens) != 'number') 
                                tokensA = this.myTokens[i].tokens.toNumber();
                            else
                                tokensA = this.myTokens[i].tokens;


                            if(typeof(this.myTokens[j].tokens) != 'number') 
                                tokensB = this.myTokens[j].tokens.toNumber();
                            else
                                tokensB = this.myTokens[j].tokens;

                            let sumTokens = tokensA + tokensB;
                            this.myTokens[i].tokens = sumTokens;
                            this.myTokens[j].id = 'revised';
                        }
                    }
                    if (j == this.myTokens.length)
                    {
                        for (j = 0; j < this.myTokens.length; j++)
                        {
                            if(this.myTokens[j].id == 'revised'){
                                this.myTokens.splice(j, 1);
                            }
                        }
                    }
                }

                for (let i = 0; i < this.myTokens.length; i++)
                {
                    this.totalUserTokens += this.myTokens[i].tokens;
                }

                this.tokensRender = true;

            } catch (err) {
                console.log(err);
            }
        },

        pause() {
            const iframes = document.querySelectorAll("#modal iframe");
            iframes.forEach(element => {
                const srcArray = element.src.split("");
                srcArray[srcArray.length - 1] = "0";
                const newSrc = srcArray.join("");
                element.src = newSrc;
            });
        }
    }, 
    
    mounted() {
        /**
        * Easy selector helper function
        */
        const select = (el, all = false) => {
        el = el.trim()
        if (all) {
            return [...document.querySelectorAll(el)]
        } else {
            return document.querySelector(el)
        }
        }

        /**
        * Easy event listener function
        */
        const on = (type, el, listener, all = false) => {
        let selectEl = select(el, all)
        if (selectEl) {
            if (all) {
            selectEl.forEach(e => e.addEventListener(type, listener))
            } else {
            selectEl.addEventListener(type, listener)
            }
        }
        }

        /**
        * Easy on scroll event listener 
        */
        const onscroll = (el, listener) => {
        el.addEventListener('scroll', listener)
        }

        /**
        * Navbar links active state on scroll
        */
        let navbarlinks = select('#navbar .scrollto', true)
        const navbarlinksActive = () => {
        let position = window.scrollY + 200
        navbarlinks.forEach(navbarlink => {
            if (!navbarlink.hash) return
            let section = select(navbarlink.hash)
            if (!section) return
            if (position >= section.offsetTop && position <= (section.offsetTop + section.offsetHeight)) {
            navbarlink.classList.add('active')
            } else {
            navbarlink.classList.remove('active')
            }
        })
        }
        window.addEventListener('load', navbarlinksActive)
        onscroll(document, navbarlinksActive)

        /**
        * Scrolls to an element with header offset
        */
        const scrollto = (el) => {
        let header = select('#header')
        let offset = header.offsetHeight

        if (!header.classList.contains('header-scrolled')) {
            offset -= 20
        }

        let elementPos = select(el).offsetTop
        window.scrollTo({
            top: elementPos - offset,
            behavior: 'smooth'
        })
        }

        /** * Toggle .header-scrolled class to #header when page is scrolled
        */
        let selectHeader = select('#header')
        if (selectHeader) {
        const headerScrolled = () => {
            if (window.scrollY > 100) {
            selectHeader.classList.add('header-scrolled')
            } else {
            selectHeader.classList.remove('header-scrolled')
            }
        }
        window.addEventListener('load', headerScrolled)
        onscroll(document, headerScrolled)
        }

        /**
        * Back to top button
        */
        let backtotop = select('.back-to-top')
        if (backtotop) {
        const toggleBacktotop = () => {
            if (window.scrollY > 100) {
            backtotop.classList.add('active')
            } else {
            backtotop.classList.remove('active')
            }
        }
        window.addEventListener('load', toggleBacktotop)
        onscroll(document, toggleBacktotop)
        }

        /**
        * Mobile nav toggle
        */
        on('click', '.mobile-nav-toggle', function(e) {
        select('#navbar').classList.toggle('navbar-mobile')
        this.classList.toggle('bi-list')
        this.classList.toggle('bi-x')
        })

        /**
        * Mobile nav dropdowns activate
        */
        on('click', '.navbar .dropdown > a', function(e) {
        if (select('#navbar').classList.contains('navbar-mobile')) {
            e.preventDefault()
            this.nextElementSibling.classList.toggle('dropdown-active')
        }
        }, true)

        /**
        * Scrool with ofset on links with a class name .scrollto
        */
        on('click', '.scrollto', function(e) {
        if (select(this.hash)) {
            e.preventDefault()

            let navbar = select('#navbar')
            if (navbar.classList.contains('navbar-mobile')) {
            navbar.classList.remove('navbar-mobile')
            let navbarToggle = select('.mobile-nav-toggle')
            navbarToggle.classList.toggle('bi-list')
            navbarToggle.classList.toggle('bi-x')
            }
            scrollto(this.hash)
        }
        }, true)

        /**
        * Scroll with ofset on page load with hash links in the url
        */
        window.addEventListener('load', () => {
        if (window.location.hash) {
            if (select(window.location.hash)) {
            scrollto(window.location.hash)
            }
        }
        });
    }
}
</script>