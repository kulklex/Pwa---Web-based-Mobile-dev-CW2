<template>
    <div id="app" class="container">
        <h1 class="d-flex justify-content-center pt-4 pb-2">
            Book Available Lessons
        </h1>


        <!-- Cart Button -->
        <div class="container-fluid m-4">
            <button class="btn btn-outline-primary" v-show="cart.length" @click="changeToCartPage">
                <div class="text-center m-2" style="width: 10rem">
                    <p style="font-size: larger;">Cart <i class="fa fa-shopping-cart" aria-hidden="true"></i></p>
                    <span>{{ (cart?.length) }} items</span>
                </div>
            </button>
        </div>
        <!-- Cart Button ends-->




        <!-- Search -->
        <div class="d-flex justify-content-center">
            <form>
                <input type="text" placeholder="search" @input="searchLessons" v-model="searchInput">
            </form>
        </div>
        <!-- Search ends-->

        <!-- Sorting starts-->
        <div class="p-2">
            <div class="d-flex justify-content-center m-2">
                <button class="btn btn-outline-primary m-1" @click="sortSubjects">
                    Subject
                </button>
                <button class="btn btn-outline-primary m-1" @click="sortLocations">
                    Location
                </button>
                <button class="btn btn-outline-primary m-1" @click="sortPrices">
                    Price
                </button>
                <button class="btn btn-outline-primary m-1" @click="sortSpaces">
                    Spaces
                </button>
            </div>
            <div class="d-flex justify-content-center m-2">
                <button class="btn btn-outline-primary m-1" @click="changeToAscending" :class="{ 'btn-primary': activeBtn === 'asc' }">
                    <p>Ascending</p>
                </button>
                <button class="btn btn-outline-primary m-1" @click="changeToDescending" :class="{ 'btn-primary': activeBtn === 'dsc' }">
                    <p>Descending</p>
                </button>
                <button class="btn btn-outline-primary m-1" @click="toggleTestConsole">
                    <p>Test Console</p>
                </button>
            </div>
             <div v-if="testConsole" class="d-flex justify-content-center m-2">
                <button class="btn btn-outline-primary m-1" @click="reload">Reload Page</button>
                <button class="btn btn-outline-primary m-1" @click="delateAllCache">Delete all cache</button> 
                <button class="btn btn-outline-primary m-1" @click="allowPermission">Allow permission</button>
                <button class="btn btn-outline-primary m-1" @click="unregisterSW">Unregister All Service Workers</button>
            </div>
        </div>
        <!-- Sorting ends-->

       
        <LessonsComponent v-if="!cartPage" :lessons="lessons" @add="addToCart" />
        <CheckoutComponent v-else :checkout="checkout" :cart="cart" :cartPage="cartPage" @remove="removeFromCart" />
    </div>
</template>

<script>
import LessonsComponent from "./components/LessonsComponent.vue"
import CheckoutComponent from "./components/CheckoutComponent.vue"

export default {
    name: 'App',
    components: {
        LessonsComponent,
        CheckoutComponent
    },
    data() {
        return {
            cartPage: false,
            ascending: true,
            descending: false,
            activeBtn: "",
            cart: [],
            checkout: {
                name: "",
                phoneNumber: "",
                lessonId: null,
                numOfSpaces: 0
            },
            lessons: [],
            searchInput: "",
            testConsole: false
        }
    },
    methods: {
        async fetchLessons() {
            try {
                const response = await fetch('https://courseworkvuejs-env.eba-igkjeup4.eu-west-2.elasticbeanstalk.com/lessons');
                if (!response.ok) {
                    throw new Error('Failed to fetch lessons');
                }
                this.lessons = await response.json();
            } catch (error) {
                console.error(error);
            }
        },
        async updateLessonSpaces(lessonId, newSpaces) {
            try {
                const response = await fetch(`https://courseworkvuejs-env.eba-igkjeup4.eu-west-2.elasticbeanstalk.com/lessons/${lessonId}`, {
                    method: 'PUT',
                    body: JSON.stringify([{ _id: lessonId, spaces: newSpaces }]),
                    headers: {
                        'Content-Type': 'application/json',
                    },
                });

                return await response.json();
            } catch (error) {
                console.error(error);
                throw new Error('Failed to update lesson spaces');
            }
        },

        async addToCart(lesson) {
            this.updateLessonSpaces(lesson._id, lesson.spaces - 1)
                .then(() => {
                    lesson.spaces--;
                    if (lesson.spaces >= 0) {
                        this.cart.push(lesson);
                    }
                })
                .catch(error => {
                    console.error(error);
                });
        },
        async removeFromCart(lesson) {
            try {
                 // Increase the spaces for the lesson
                 await this.updateLessonSpaces(lesson._id, lesson.spaces + 1);

                // Update the lesson in the lessons array
                const lessonInLessonsIndex = this.lessons.findIndex(l => l._id === lesson._id);
                if (lessonInLessonsIndex !== -1) {
                    this.lessons[lessonInLessonsIndex].spaces++;
                }

                // Remove the lesson from the cart
                const lessonIndexInCart = this.cart.findIndex(cartLesson => cartLesson._id === lesson._id);
                if (lessonIndexInCart !== -1) {
                    this.cart.splice(lessonIndexInCart, 1);
                }
            } catch (error) {
                console.error(error);
            }
        },
        async createOrder() {
            try {
                // Creating the order object
                const order = {
                    name: this.checkout.name,
                    phoneNumber: this.checkout.phoneNumber,
                    lessonIDs: this.cart.map(lesson => lesson.id),
                    numOfSpaces: this.cart.map(lesson => lesson.spaces),
                };

                // Sending a POST request to create a new order
                const response = await fetch('https://courseworkvuejs-env.eba-igkjeup4.eu-west-2.elasticbeanstalk.com/orders', {
                    method: 'POST',
                    headers: {
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify(order),
                });

                const result = await response.json();
                this.message = result.msg;
                alert('You have successfully checked out!')
                window.location.href = "https://kulklex.github.io/Frontend-Web-based-Mobile-dev-CW2"
            } catch (error) {
                console.error(error);
                this.message = 'Error creating order';
            }
        },
        async searchLessons() {
            try {
                // Sending request for fetching lessons based on the searchInput value
                const response = await fetch(`https://courseworkvuejs-env.eba-igkjeup4.eu-west-2.elasticbeanstalk.com/lessons/search?filter=${this.searchInput}`);
                this.lessons = await response.json();
            } catch (error) {
                console.error(error);
            }
        },
        clearCart() {
            this.cart = [];
            this.cartPage = false;
        },
        changeToCartPage() {
            this.cartPage = !this.cartPage;
        },
        changeToAscending() {
            this.descending = false;
            this.ascending = true;
            this.activeBtn = 'asc'
        },
        changeToDescending() {
            this.ascending = false;
            this.descending = true;
            this.activeBtn = 'dsc'
        },
        sortPrices() {
            if (this.ascending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    return a.price - b.price;
                })
            }
            if (this.descending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    return b.price - a.price
                })
            }
        },
        sortLocations() {
            if (this.ascending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    let subsA = a.location.toLowerCase()
                    let subsB = b.location.toLowerCase()
                    if (subsA < subsB) {
                        return -1
                    }
                    if (subsA > subsB) {
                        return 1
                    }
                    return 0
                })
            }
            if (this.descending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    let subsA = a.location.toLowerCase()
                    let subsB = b.location.toLowerCase()
                    if (subsA < subsB) {
                        return 1
                    }
                    if (subsA > subsB) {
                        return -1
                    }
                    return 0
                })
            }
        },
        sortSpaces() {
            if (this.ascending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    return a.spaces - b.spaces;
                })
            }
            if (this.descending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    return b.spaces - a.spaces;
                })
            }
        },
        sortSubjects() {
            if (this.ascending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    let subsA = a.subject.toLowerCase()
                    let subsB = b.subject.toLowerCase()
                    if (subsA < subsB) {
                        return -1
                    }
                    if (subsA > subsB) {
                        return 1
                    }
                    return 0
                })
            }
            if (this.descending) {
                let sortedSubs = this.lessons
                sortedSubs = sortedSubs.sort((a, b) => {
                    let subsA = a.subject.toLowerCase()
                    let subsB = b.subject.toLowerCase()
                    if (subsA < subsB) {
                        return 1
                    }
                    if (subsA > subsB) {
                        return -1
                    }
                    return 0
                })
            }
        },
        isLetters($event) {
            var charCode = $event.keyCode;
            if ((charCode > 64 && charCode < 91) || (charCode > 96 && charCode < 123) || charCode == 8) {
                return true;
            } else {
                $event.preventDefault();
            }
        },
        isNumbers($event) {
            var charCode = $event.keyCode;
            if ((charCode > 31 && (charCode < 48 || charCode > 57))) {
                $event.preventDefault();
            } else {
                return true;
            }
        },
        reload() {
            window.location.reload(true);
        },
        allowPermission() {
            window.open('https://courseworkvuejs-env.eba-igkjeup4.eu-west-2.elasticbeanstalk.com/lessons', '_blank');
        },
        delateAllCache() {
            caches.keys().then(function (names) {
                for (let name of names)
                    caches.delete(name);
            });
            console.log("All Caches Deleted");
        },
        unregisterSW() {
            navigator.serviceWorker.getRegistrations().then(function (registrations) {
                    for (let registration of registrations) {
                        registration.unregister()
                    }
                });
                console.log("ServiceWorkers Unregistered");
        },
        toggleTestConsole() {
            this.testConsole = !this.testConsole
        }
    },
    mounted() {
        // Fetch lessons when the app is mounted
        this.fetchLessons();
    },
    // Watch for changes in the 'searchInput' data property
    watch: {
        // When 'searchInput' changes, call the searchLessons method
        searchInput: function () {
            this.searchLessons();
        },
    },
   created: function () {
        if ("serviceWorker" in navigator) {
            navigator.serviceWorker.register("service-worker.js");
        }
    }
}
</script>

<style>
#app {
    font-family: Avenir, Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
}
</style>
