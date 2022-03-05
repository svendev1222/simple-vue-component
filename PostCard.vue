<template>
    <div class="my-3 card hover-shadow-3 bg-white" :id="'post-' + post.id">
        <div class="card-body">
            <div class="d-flex m-0 justify-content-between">
                <div class="">
                    #{{post.id}}
                </div>
                <div class="">
                    <a :href="'/posts/'+post.id"  target="_blank" data-toggle="tooltip" data-placement="left" title="Open post in new window"><i class="far fa-external-link fa-1x"></i></a>
                </div>
            </div>
            <div class="my-1 d-flex justify-content-between flex-md-row flex-column">
                <div class="">
                    <!--                    <div class="square-72 d-block mr-8">-->
                    <!--                        &lt;!&ndash;                            <img src="./image/l2/png/featured-job-logo-1.png" alt="">&ndash;&gt;-->
                    <!--                    </div>-->
                    <div>
                        <h3 class="font-weight-bolder post-title"><a class="font-weight-bolder" href="">{{ post.title }}</a></h3>
                        <p class="mt-1 text-muted company">{{ post.company_name }}</p>
                    </div>
                </div>
                <div class="d-flex align-items-baseline salary">
                    <div class="mr-2">
                        <i class="fas fa-hand-holding-usd fa-2x"></i>
                    </div>
                    <p class="font-size-7 font-weight-bold">
                        {{salary(post.income_range[0])}}-{{salary(post.income_range[1])}}K <span>USD</span>
                    </p>
                </div>
            </div>
            <div class="row">
                <div class="col-md-6">
                    <ul class="d-flex list-unstyled mr-n3 flex-wrap">
                        <li class="m-1" v-for="highlight in highlights">
                            <div class="p-2 highlight rounded">
                                {{highlight}}
                            </div>
                        </li>
                    </ul>
                </div>
                <div class="col-md-6 align-self-center">
                    <ul class="d-flex list-unstyled mr-n3 flex-wrap mr-n8 justify-content-md-end post-details">
                        <li class="mt-2 mr-4 d-flex" v-if="isLocation">
                            <span class="mr-2"><i class="far fa-map-marker-alt"></i></span>
                            <span class="font-weight-semibold">{{ post.location.city }}, {{ post.location.state }} <span v-if="calcDistance">({{calcDistance}}mi)</span></span>
                        </li>
                        <li class="mt-2 mr-4 d-flex">
                            <span class="mr-2"><i class="far fa-suitcase"></i></span>
                            <span class="font-weight-semibold">{{ duration }}</span>
                        </li>
                        <li class="mt-2 mr-4 d-flex">
                            <span class="mr-2"><i class="far fa-clock"></i></span>
                            <span class="font-weight-semibold">{{ postDate }}</span>
                        </li>
                    </ul>
                </div>
            </div>
            <div class="d-flex justify-content-center row-post">
                <a @click.prevent="clickPostCard(post.id)" :href="'/posts/'+post.id" class="btn btn-outline-primary">View Post</a>
                <a @click.prevent="wishList(post.id)" class="btn-wish">
                    <template v-if="isLoggedIn">
                        <i :class="[ isLoading ? 'fas fa-spin fa-spinner fa-2x' : isLike? 'fa fa-star fa-2x' : 'far fa-star fa-2x' ]"></i>
                    </template>
                    <template v-else>
                        <i class="fal fa-star fa-2x"></i>
                    </template>
                </a>
            </div>
        </div>
    </div>
</template>

<script>
// import moment from 'moment';
import Distance from "../../../../utils/Distance";
import ScreenHelper from "../../../../services/ScreenHelper";
import {computed, onMounted, onUnmounted, ref, toRefs} from 'vue';
import Common from "../../../../../../js/utils/Common";

import FavouriteService from "../../../../../../userApp/js/services/FavouriteService";

import axios from "axios";
// import Notification from "../../../../components/notification";
export default {
    name: "PostCard",
    components: {Notification},
    props: {
        'post': Object,
        'filterOpts': Object,
    },
    data() {
        return {
            isLoading: false,
            isLoggedIn: false,
            isLike: false,
        }
    },
    mounted() {
      if(this.$store.getters['posts/isLoggedIn']){
          this.isLoggedIn = true;
      }
      this.getPosts()
    },
    methods: {
        async getPosts() {
            await FavouriteService.loadFavourites()
            const favorites = FavouriteService.favourites;
            const tmp = favorites.filter(x => x.id == this.post.id)
            this.isLike = tmp.length != 0;
            

        },
         wishList(postId) {
             console.log(postId)
            if(this.$store.getters['posts/isLoggedIn']) {
                this.isLoading = true;
                // this.isLike = true;
                axios.post('/wishlist', {
                    postId: postId
                })
                    .then((res) => {
                        this.isLoading = false;
                        if(res.data.msg === 'success'){
                            this.isLike = true;

                            this.$toast.success('Success', {
                                position: 'bottom-right',
                                duration: 5000,
                            });
                        }else{
                            this.$toast.error('You can not insert duplicated row',{
                                position: 'bottom-right',
                                duration: 5000,
                            });
                        }
                    })
                    .catch((error) => {
                        this.isLoading = false;
                        this.$toast.error('Error');
                        console.log(error);
                    });
            } else {
                window.location.href='/login';
            }
        }
    },
    setup(props, {emit}) {
        const mobileView = ref(false);
        const show = ref(false);
        const {post, filterOpts} = toRefs(props);
        const posts = ref([]);

        const calcDistance = computed(() => {
            if (!post.value.location.latitude ||
                !post.value.location.longitude ||
                !filterOpts.value.location.distance.latitude ||
                !filterOpts.value.location.distance.longitude) {
                return;
            }
            return Distance.calc(post.value.location.latitude, post.value.location.longitude, filterOpts.value.location.distance.latitude, filterOpts.value.location.distance.longitude);
        });

        function salary(amount) {
            if (!isNaN(amount)) {
                return amount / 1000;
            }
        }

        const postDate = computed(()=>{
            return dayjs(post.value.updated_at).fromNow();
        })

        const isLocation = computed(()=>{
            if (!post.value.location) {
                return false;
            }
            let location = Object.values(post.value.location);
            let emptyVal = 0;
            for (let i = 0; i < location.length; i++) {
                if (!location[i] || location[i] === null || location[i].length === 0) {
                    emptyVal++;
                }
            }
            return emptyVal <= 3;
        });

        const highlights = computed(()=>{
            let items = [];
            if (post.value.fellowship_selected?.length > 0) {
                items.push('Fellowship');
            }
            if (post.value.employment_type) {
                items.push(post.value.employment_type.toUpperCase());
            }
            if (post.value.practice_type) {
                items.push(Common().capitalize(post.value.practice_type) + ' Practice');
            }
            if (post.value.duration?.length > 0) {
                post.value.duration.forEach((duration) => {
                    items.push(Common().camelCaseToTitleCase(duration))
                })
            }
            if (post.value.start_date === 'immediately') {
                items.push('Immediate Hire');
            }
            return items;
        });


        const duration = computed(()=>{
            if (post.value.duration?.length > 0) {
                if (post.value.duration.indexOf('fullTime') !== -1) {
                    return 'Full-time';
                }
                else if (post.value.duration.indexOf('partTime') !== -1) {
                    return 'Part-time';
                }
                else if (post.value.duration.indexOf('locum') !== -1) {
                    return 'Locum';
                }

            } else {
                return 'Full-time';
            }
        })

        onMounted(() => {
            ScreenHelper.subscribe(function () {
                checkMobileView()
            });
            checkMobileView();

        });

        // onMounted(() => {
        //     getPosts();
        // })


        onUnmounted(() => {
            ScreenHelper.unsubscribe(function () {
                checkMobileView()
            });
        })

        const checkMobileView = () => {
            mobileView.value = ScreenHelper.isMobileView;
            // this.$forceUpdate();
            // console.log(mobileView.value);
        }

        const clickPostCard = (postId) => {
            emit("viewPost", postId);
            location.href = '/posts/' + postId;
        }


        // async function getPosts() {
        //     await FavouriteService.loadFavourites()
        //     const favorites = FavouriteService.favourites;
        //     const tmp = favorites.filter(x => x.id == this.post.id)
        //     this.isLike = tmp.length != 0;
            

        // }


        return {
            mobileView,
            post,
            postDate,
            isLocation,
            salary,
            calcDistance,
            highlights,
            duration,
            clickPostCard,

        }
    },

};
</script>
<style scoped>
.row-post{
    position:relative;
}
.btn-wish{
    position:absolute;
    right: 0;
    color: red;
    cursor:pointer;
}



@media (max-width: 767px) {
    /*<div class="row post-info mb-2">*/
    div.post-info, div > span > ul.list-unstyled {
        text-align: center
    }

}

.smooth-enter-active, .smooth-leave-active {
    transition: height 1s;
    overflow: hidden;
}

.smooth-enter, .smooth-leave-to {
    height: 0;
}

.card {
    background: #fff;
    padding: 10px;
    /*box-shadow: 0 0 10px #d8d8d8;*/
    box-shadow: 5px 5px 5px #bbb;
    position: relative;
    margin-bottom: 30px;
    border-radius: .75rem;
}

/*div.salary i {*/
/*    color: var(--main-site-color);*/
/*}*/

/*.font-size-7 {*/
/*    font-size: 1.75rem;*/
/*    line-height: 1.5;*/
/*    letter-spacing: 1px;*/
/*}*/

/*.salary p {*/
/*    color: var(--name-txt-color);*/
/*}*/

/*.salary p span {*/
/*    color: var(--info-txt-color);*/
/*}*/

/*h3 a {*/
/*    color: var(--schema-color-5);*/
/*}*/

/*h3 a:hover {*/
/*    color: var(--main-site-color);*/
/*}*/

/*.post-title {*/
/*    letter-spacing: 1px;*/
/*}*/

/*.company {*/
/*    font-size: .90rem;*/
/*    letter-spacing: 1px;*/
/*}*/

.card-id {
    font-size: small;
}

/*.highlight {*/
/*    background-color: #dee2e6;*/
/*    font-size: 13px;*/
/*    line-height: 2;*/
/*    letter-spacing: 0.26px;*/
/*    text-align: center !important;*/
/*    min-width: 100px;*/
/*}*/

/*.post-details {*/
/*    font-size: 15px;*/
/*}*/

/*.post-details .font-weight-semibold {*/
/*    font-weight: 500;*/
/*}*/

</style>
