<template>
  <div id="bev-tax-app">
    <div class="search">
      <input
        id="search-bar"
        v-model="search"
        class="search-field"
        type="text"
        placeholder="Search by distributor name, address, or phone #"
        @keyup.enter="filter()"
      >
      <input
        ref="archive-search-bar"
        type="submit"
        class="search-submit"
        value="Search"
        @click="filter()"
      >
    </div>
  
    <div
      v-show="loading"
      class="mtm center"
    >
      <i class="fas fa-spinner fa-spin fa-3x" />
    </div>
    <div
      v-show="!loading && emptyResponse"
      class="h3 mtm center"
    >
      Sorry, there are no results.
    </div>
    <div
      v-show="failure"
      class="h3 mtm center"
    >
      Sorry, there was a problem. Please try again.
    </div>
    <div
      v-if="!loading && !emptyResponse && !failure"
      class="table-container"
    >
      <table
        v-if="filteredDistributors.length"
      >
        <thead
          class="center"
          data-sticky
          data-top-anchor="filter-results:bottom"
          data-btm-anchor="page:bottom"
        >
          <tr>
            <th
              class="dist-name"
            >
              <span>Distributor Name</span>
            </th>
            <th
              class="dist-address"
            >
              <span>Address</span>
            </th>
            <th
              class="dist-contact"
            >
              <span>Contact Information</span>
            </th>
          </tr>
        </thead>
        <!-- <tbody> -->
        <paginate
          ref="paginator"
          name="filteredDistributors"
          :list="filteredDistributors"
          class="paginate-list"
          tag="tbody"
          :per="50"
        >
          <tr
            v-for="distributor in paginated('filteredDistributors')"
            :key="distributor.cartodb_id"
          >
            <td
              class="business-name"
            >
              {{ distributor.doing_business_as_name | upperCase }}
            </td>
            <td>
              <a
                v-if="distributor.street_address"
                :href="'https://www.google.com/maps/search/?api=1&query=' + distributor.street_address + ' ' + distributor.zip_code"
                target="_blank"
                class="external"
              >
                {{ distributor.street_address }} <br>
                {{ distributor.city }}, {{ distributor.state }}
                {{ distributor.zip_code }}
              </a>
            </td>
            <td>
              <a
                :href="'tel:' + distributor.phone_number"
                class="telephone"
              >
                {{ distributor.phone_number | phoneDisplay }}
              </a>
              <!-- check to see if there is an extension -->
              <span 
                v-if="distributor.phone_extension !== '0' && distributor.phone_extension "
              >
                <b>ext. {{ distributor.phone_extension }}</b>
              </span>
              <br>
              <br>
              <!-- culls for email addresses  -->
              <a 
                v-if="distributor.website !== null && !distributor.website.includes('@')" 
                target="_blank" 
                :href="toLink(distributor.website)"
                class="external"
              > 
                {{ toLink(distributor.website) }}
              </a>
            </td>
          </tr>
        </paginate>
        <!-- </tbody> -->
      </table>
      <div class="app-pages">
        <p> Showing <b> {{ numberOf }} </b> distributors </p>
        <paginate-links
          v-show="!loading && !emptyResponse && !failure"
          for="filteredDistributors"
          :async="true"
          :limit="3"
          :show-step-links="true"
          :hide-single-page="false"
          :step-links="{
            next: 'Next',
            prev: 'Previous'
          }"
          @change="scrollToTop"
        />
      </div>
    </div>
  </div>
</template>
<script>

import Vue from "vue";
import axios from "axios";
import moment from "moment";
import VuePaginate from "vue-paginate";
import VueFuse from "vue-fuse";

Vue.use(VuePaginate);
Vue.use(VueFuse);

const endpoint = "https://phl.carto.com/api/v2/sql";
const query = "?q=SELECT%20*%20FROM%20beverage_tax_registration_data";

export default {
  name: "PhillyBevTaxDistributors",
  components: { 

  },
  filters: {
    'phoneDisplay' : function(val) {
      if(val !== null) {
        return val.replace(/[^0-9]/g, '')
          .replace(/(\d{3})(\d{3})(\d{4})/, '($1) $2-$3');
      }
    },

    'upperCase': function(val) {
      if (val) {
        return val.toUpperCase().trim();
      }
    },
  },
  data: function() {
    return {
      distributors : [],
      filteredDistributors: [],
      sortedDistributors: [],
      loading : true,
      emptyResponse : false,
      failure : false,
      displayPaginate: true,
      search: '',
      searchOptions: {
        shouldSort: false,
        threshold: 0.2,
        keys: [
          "doing_business_as_name",
          "street_address",
          "city",
          "state",
          "zip_code",
          "phone_number",
        ],
      },
      paginate: [ "filteredDistributors" ],
    };
  },
  computed: { 
    numberOf: function() {
      return this.filteredDistributors.length;
    },
  },

  watch: {
    search: function() {
      this.filter();
    },
  },

  created:  function() {
    this.getDistributors();
  },

  methods: {
    getDistributors: function() {
      {
        axios.get(endpoint+query)
          .then(response => {
            this.distributors = response.data.rows;

            //sort alphabetically
            this.sortedDistributors = this.distributors.sort(function(a, b){
              if(a.doing_business_as_name.toLowerCase() < b.doing_business_as_name.toLowerCase()) {
                return -1; 
              }
              if(a.doing_business_as_name.toLowerCase() > b.doing_business_as_name.toLowerCase()) {
                return 1; 
              }
              return 0;
            });

            this.filteredDistributors = this.sortedDistributors;
            this.loading = false;
          }).catch(e => {
            window.console.log(e);
            this.failure = true;
            this.loading = false;
          });
      }
    },


    filterSearch: function() {
      if (this.search !== '') { // there is nothing in the search bar -> return everything in filteredPosts
        this.filteredDistributors = [];
        
        this.$search(this.search, this.distributors, this.searchOptions).then(posts => {
          this.filteredDistributors = posts;
        });
    
      } else {
        this.filteredDistributors = this.sortedDistributors;
    
      }

    },

    filter: async function() {

      await this.filterSearch();
     
      await this.checkEmpty();
      if (this.$refs.paginator) {
        this.$refs.paginator.goToPage(1);
      }
    },

    checkEmpty: function() {
      this.emptyResponse = (this.filteredDistributors.length === 0 ? true : false);
      // this.displayPaginate = (this.filteredDistributors.length >= 50) ? true : false;
    },
    
    scrollToTop : function () {
      window.scrollTo({
        top: 0,
        behavior: 'smooth',
      });
    },

    toLink(s) {
      var prefix = 'http://';
      if (s.substr(0, prefix.length) !== prefix) {
        return s = prefix + s;
      }
    },
  },
};
</script>

<style lang="scss">

#bev-tax-app {
  width: 80%;
  margin: 10px auto;
  max-width: 1024px;

  .business-name {
    font-weight: bold;
    font-size: 16px;
  }

  .dist-name{
    width: 40%;
  }

  .dist-address {
    width: 30%;
  }

  .dist-contact {
    width: 30%;
  }

  .app-pages {
    display: flex;
    justify-content: space-between;
  }
}
</style>