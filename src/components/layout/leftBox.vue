<template>
  	<div id="leftBox" ref="leftBox" class="col-lg-9 col-md-8 col-sm-12">
        <!-- 文章 -->
        <div v-if="article.length >= 1" :class="imgLoadStatus?'articleList loadSucceed':'articleList'" :style="'height:'+Math.max.apply(Math,(articleHeight.length>=1?articleHeight:[0]))+'px;'">
            <article v-for="(item,index) in article" :style="imgLoadStatus?('top:'+position[index].top+'px;left:'+position[index].left+'px;z-index:'+(10-index)+';'):'top:0;left:0;z-index:'+(10-index)+';'">
                <div class="u_transition post-content-container">
                    <router-link v-if="item.images_src.length === 1" class="articleImg" :to="{path: '/articleDetail',query: {articleId: item._id,title: item.title}}">
                        <img ondragstart="return false;" @mouseover="item.imgMouseStatus=true" @mouseout="item.imgMouseStatus=false" :class="{u_transition: true,scale: item.imgMouseStatus}" :src="item.images_src" @error="imgError();"/>
                    </router-link>
                    <div v-else class="swiper-container">
                        <swiper :options="swiperOption[index]" ref="mySwiper">
                            <swiper-slide v-for="banner in item.images_src">
                                <img style="width: 100%;" ondragstart="return false;" class="swiper-lazy" :data-src="banner" @error="imgError();"/>
                                <div class="swiper-lazy-preloader"></div>
                            </swiper-slide>
                        </swiper>
                        <div :class="'swiper-pagination swiper-pagination'+index+' swiper-pagination-bullets'"></div>
                    </div>
                    <div class="box">
                        <h2><router-link class="u_transition u_hover_blue" :to="{path: '/articleDetail',query: {articleId: item._id,title: item.title}}">{{item.title}}</router-link></h2>
                        <strong>{{item.content}}</strong>
                        <p>
                            <b class="article_categories">
                                <span v-for="(categories,index) in item.categories">
                                    {{index == 0 ? '':', '}}
                                    <a @click="$emit('searchCnt', {type: 'Category', text: categories})" class="u_transition u_hover_blue">
                                        {{categories}}
                                    </a>
                                </span>
                            </b>
                            <router-link class="review u_transition u_hover_blue_bg" :to="{path: '/articleDetail',query: {articleId: item._id,title: item.title}}">
                                <i></i>
                                {{item.review}}
                            </router-link>
                            <time class="g-c-center">
                                <span class="g-r-center"><i></i>{{Date.parse(item.creation_at) | dateFormat('YYYY/MM/DD')}}</span>
                                <span><i></i>{{item.browsing}}</span>
                            </time>
                        </p>
                    </div>
                </div>
            </article>
        </div>
        <div class="noParam" v-else>
            <img ondragstart="return false;" src="../../images/noParam.png" alt="暂无数据" />
            <p>找不到相关 "{{searchText}}" 数据</p>
        </div>
        <!-- 分页 -->
        <div v-if="pageNum > 1 && imgLoadStatus" class="pagination g-c-center">
            <ul class="page-numbers g-c-center">
                <li v-if="page > 1" @click="pageBtn(Number(page)-1,'NoFirst')">
                    <a class="next page-numbers u_transition u_hover_blue">上一页</a>
                </li>
                <li v-for="(item,index) in pageNum" @click="pageBtn(index+1,'NoFirst')">
                    <span v-if="index + 1 == page" class="page-numbers current">{{index+1}}</span>
                    <a v-else class="page-numbers u_transition u_hover_blue">{{index+1}}</a>
                </li>
                <li v-if="pageNum > page" @click="pageBtn(Number(page)+1,'NoFirst')">
                    <a class="next page-numbers u_transition u_hover_blue">下一页</a>
                </li>
            </ul>
        </div>
  	</div>
</template>

<script>
"use strict";
import loadIng   from './loadIng.vue'

export default {
    props: ["searchData","categoriesName","resizeTime"],
    mounted() {
        //获取文章列表
        let page = this.$route.query.page ? Number(this.$route.query.page) : 1,
            that = this;

        for (var i in this.$route.query) {
            if (i == "_s" || i == "Tag" || i == "Category")
                this.searchText = this.$route.query[i];
        }

        this.getArticlesList(page,"",this.searchText);
    },
    data() {
        return {
            page: null,
            pageNum: null,
            imgLoadStatus: false,
            load_success_number: 0,
            searchText: "",
            article: [],
            articleHeight: [],
            position: {},
            swiperOption: {},
            timer: null
        }
    },
    methods: {
        /** 获取文章类别
          * @data   page: 分页, type: {hot: '最新动态',NoFirst: '区分是否是一次访问'}, searchCnt: 搜索内容
         */
        getArticlesList(page,type,searchCnt) {

            let that = this,
                imgHost = this.$store.state.IMGHOST,
                apiHost = this.$store.state.APIHOST;

            that.position = [];
            that.article = [];
            that.imgLoadStatus = false;
            that.$emit('imgLoadStatus', false);
            that.load_success_number = 0;
            that.swiperOption = {};

            that.$http.jsonp(apiHost + 'api/getArticlesList?page='+page+(that.categoriesName?'&categories='+that.categoriesName:'')+(searchCnt?'&searchCnt='+searchCnt:'')).then((res) => {
                let clientWidth = document.documentElement.clientWidth,
                    offsetWidth = that.$refs.leftBox.offsetWidth,
                    imgWidth = Math.round((clientWidth>=768)?((offsetWidth - 60) / 2):(offsetWidth - 30));

                if (res.body.code == 0) {

                    if (type == "NoFirst" || searchCnt)
                        document.getElementsByTagName('body')[0].scrollTop = 0;

                    let data = res.body.data;

                    that.page = data.current_page;
                    that.pageNum = data.last_page;

                    if(data.data.length <= 0) {
                        that.imgLoadStatus = true;
                        that.$emit('imgLoadStatus', true);
                    }

                    for (var i in data.data) {
                        let x = i;

                        data.data[i].index = Number(i);
                        data.data[i].imgMouseStatus = false;
                        data.data[i].images_src.forEach( ( item, j ) => {
                            let $img = new Image(),
                                src = imgHost + item + '?imageView2/2/w/' + imgWidth + '/interlace/1';

                            if (j === 0) {
                                data.data[i].images_src[j] = $img.src = src;
                                
                                $img.onload = () => {
                                    this.setArticleLocation({type: "load",event: event,imgHeight: $img.height,imgWidth: imgWidth,clientWidth: clientWidth},data.data);
                                }

                                $img.onerror = () => {
                                    this.setArticleLocation({type: "error",event: event,imgHeight: $img.height,imgWidth: imgWidth,clientWidth: clientWidth},data.data);
                                }
                            } else {
                                data.data[i].images_src[j] = src;

                                //swiper插件   同个页面多个滑动  动态配置
                                that.swiperOption[i] = {
                                    effect : 'coverflow',
                                    initialSlide: 0,
                                    lazyLoading : true,
                                    pagination: '.swiper-pagination' + i
                                }
                            }
                        });
                    }

                    that.article = data.data;
                }
            });
        },
        setArticleLocation(obj,param) {
            if (obj.type == "onresize") {
                this.position.forEach((item,i) => {
                    this.position[i].width = obj.imgWidth;
                    this.position[i].height = Math.round((item.height - 279) * obj.ratio) + 279;
                });
            } else {
                var imgSrc = obj.event.srcElement.src;

                //记录每张图片高度
                for (var x in param) {
                    if (imgSrc === param[x].images_src[0]) {
                        this.position[x] = {
                            width: obj.imgWidth,
                            height: (obj.type==="error" ? Number(obj.imgWidth / 2) : obj.imgHeight) + 279
                        }
                        break;
                    }
                }
            }

            //设置定位信息
            if (this.load_success_number >= param.length-1) {
                this.articleHeight = [];

                for (var y in this.position) {
                    if (y <= (obj.clientWidth>768?1:0)) {
                        this.position[y].left = y % 2 === 0 ? 0 : obj.imgWidth + 15;
                        this.position[y].top = 0;
                        this.articleHeight.push(this.position[y].height);
                    } else {
                        this.position[y].top = Math.min.apply(Math,this.articleHeight);
                        this.position[y].left = this.articleHeight[0] > this.articleHeight[1] ? obj.imgWidth + 15 : 0;
                        this.articleHeight[(this.articleHeight[0] > this.articleHeight[1] ? 1 : 0)] += this.position[y].height;
                    }
                }

                this.imgLoadStatus = true;
                this.$emit('imgLoadStatus', true);
            }

            this.load_success_number++;

        },
        pageBtn(num,type) {
            let data = {
                page: num
            }

            if (this.searchText)
                data._s = this.searchText
            
            this.$router.push({query: data});
            this.getArticlesList(num,type,this.searchText);
        },
        imgError() {
            let img = event.srcElement;
            img.src = "assets/images/image_error.png";
            img.onerror = null;
        }
    },
    watch: {
    	categoriesName(val) {
    		this.getArticlesList(1);
    	},
        searchData(val) {
            this.searchText = val.content;
            this.getArticlesList(1,"",val.content);
        },
        resizeTime(val) {
            let that = this,
                clientWidth = document.documentElement.clientWidth,
                offsetWidth = that.$refs.leftBox.offsetWidth,
                imgWidth = Math.round((clientWidth>=768)?((offsetWidth - 60) / 2):(offsetWidth - 30));

            clearTimeout(that.timer);

            that.timer = setTimeout(function() {
                that.setArticleLocation({type: "onresize",imgWidth: imgWidth,clientWidth: clientWidth,ratio: imgWidth/that.position[0].width},that.article);
            },100);
        }
    },
    components: {
        loadIng
    }
}
</script>

<style lang="scss" scoped>
    #leftBox {
        padding-bottom: 20px;
    }
    .articleList {
        position: relative;
        width: 100%;
        opacity: 0;
        &.loadSucceed {
            opacity: 1;
            article {
                width: 50%;
                transition: all .5s ease;
                -moz-transition: all .5s ease;  /* Firefox 4 */
                -webkit-transition: all .5s ease;   /* Safari 和 Chrome */
                -o-transition: all .5s ease;    /* Opera */
            }
            
        }
        article {
            position: absolute;
            padding-right: 15px;
            .articleImg {
                display: block;
                width: 100%;
                overflow: hidden;
                cursor: pointer;
                img {
                    width: 100% !important;
                    &.scale {
                        transform: scale(1.2);
                        -webkit-transform: scale(1.2);
                        -moz-transform: scale(1.2);
                        -o-transform: scale(1.2);
                    }
                }
            }
            
            .box {
                height: 264px;
                padding-top: 20px;
                overflow-y: hidden;
                background-color: #fff;
                h2, strong {
                    width: 100%;
                    height: 58px;
                    padding: 0 13%;
                    display: -webkit-box;
                    text-overflow: ellipsis;
                    overflow: hidden;
                    word-break: break-all;                
                    -webkit-box-orient: vertical;
                }
                h2 {
                    font-size: 28px;
                    margin: 12px 0 14px;
                    line-height: 100%!important;
                    -webkit-line-clamp: 2;
                    a {
                        color: #000;
                    }
                }
                strong {
                    margin-bottom: 28px;
                    color: #8d8d8d;
                    text-indent: 1em;
                    font-weight: 400;
                    font-size: 14px;
                    -webkit-line-clamp: 3;
                }
                p {
                    display: flex;
                    display: -webkit-flex;
                    flex: 1;
                    position: relative;
                    padding: 25px 0 48px 13%;
                    margin-right: 40%;
                    border-top: 1px solid #e9e9e9;
                    .article_categories {
                        margin-right: 15px;
                         font-weight: normal;
                         a {
                            line-height: 20px;
                            color: #8d8d8d;
                            font-size: 13px;
                         }
                    }
                    time{
                        position: absolute;
                        right: -66%;
                        width: 67%;
                        padding-right: 20px;
                        color: #8d8d8d !important;
                        align-items: flex-end;
                        i { padding-right: 5px; }
                    }
                    .review {
                        position: absolute;
                        top: -20px;
                        right: -45%;
                        width: 35%;
                        height: 40px;
                        text-align: center;
                        line-height: 40px;
                        color: #8d8d8d;
                        background-color: #f6f6f6;
                        border-radius: 100px;
                        &.u_hover_blue_bg:hover {
                            color: #fff;
                        }
                    }
                }
            }
        }
    }

    .noParam {
        padding-top: 50px;
        text-align: center;
        img {
            width: 30%;
        }
        p {
            line-height: 50px;
            font-size: 20px;
            color: #8d8d8d;
        }
    }

    .pagination {
        margin: 20px 0 0 0;
        width: 100%;
        ul {
            display: -webkit-box;
            margin: 0 auto;
            list-style: none;
            background-color: #f4f4f4;
            padding: 5px 4px;
            border-radius: 30px;
            border-width: 0px;
            li {
                margin: 0 2px;
                span,a {
                    display: block;
                    height: 40px;
                    width: 40px;
                    text-align: center;
                    line-height: 40px;
                    color: #8d8d8d;
                    cursor: pointer;
                }
                a.next {
                    width: 60px;
                }
                span {
                    background-color: #47c9e5;
                    border-radius: 30px;
                    color: #fff;
                }
            }
        }
    }
</style>