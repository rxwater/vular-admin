<template>
  <div class="d-flex flex-row align-center flex-1">
    <v-checkbox
      :label="selectedCounts > 0 ? $t('list.selected-counts').replace('{0}', selectedCounts) :  $t('list.select-all')"
      class ="mt-5"
      v-model = "selectedSome"
      :indeterminate = "selectedCounts > 0 && selectedCounts != inputValue.rows.length"
      @change = "onSelectedChange"
    ></v-checkbox>
    <v-spacer></v-spacer>
    <template v-if="!selectedSome">
      <div 
        :style="{width: searchboxWidth + 'px'}"
        class="list-search-box"
      >
        <v-text-field
          hide-details
          prepend-inner-icon="mdi-magnify"
          style="padding-top:0px;"
          class="mt-n1"
          v-model="keywords"
          dense
          :clearable = "!!inputValue.formModel.keywords"
          @click:clear = "onClearKeywords"
          @focus="searchBoxFocused = true"
          @blur = "searchBoxFocused = false"
          @keyup.13 = "onSearch"
        ></v-text-field>
      </div>
      <div v-if="!collopsed">
        <template 
          v-for="(filter, index) in shortcutFilters"
        >
          {{filter.title}}
          <v-menu offset-y
            :key="index"
          >
            <template v-slot:activator="{ on }">
              <v-btn
                :color="inputValue.formModel[filter.field] === filter.blankValue ? '' : 'primary'"
                class="ml-2 mr-3 filter-button"
                outlined
                rounded
                :small="isStick"
                v-on="on"
              >
                {{filter.blankValue === inputValue.formModel[filter.field] ? filter.blankTitle : filter.rules[inputValue.formModel[filter.field]]}}
                <v-icon right dark>mdi-chevron-down</v-icon>
              </v-btn>
            </template>
            <v-list>
              <v-list-item link color="primary" 
                
                @click = "onFilter(filter, filter.blankValue)"
              >
                <v-list-item-content>
                  <v-list-item-title>{{filter.blankTitle}}</v-list-item-title>
                </v-list-item-content>
              </v-list-item>
              <v-list-item link color="primary"
                v-for="(label, value) in filter.rules"
                :key = 'value'
                
                @click = "onFilter(filter, value)"
              >
                <v-list-item-content>
                  <v-list-item-title>{{label}}</v-list-item-title>
                </v-list-item-content>
              </v-list-item>
            </v-list>
          </v-menu>
        </template>
      </div>
      <v-menu offset-y 
        :close-on-content-click="false" 
        :nudge-width="150"
        v-model="isPopedFilters"
        v-if="popFilters.length > 0"
        >
        <template v-slot:activator="{ on }">
          <v-btn icon fab 
            v-on="on"
            :color="popFilersSelected ? 'primary' : ''"
            :small="!isStick"
            :x-small="isStick"
          >
            <v-icon class="top-small-button">mdi-dots-horizontal</v-icon>
          </v-btn>
        </template>
        <v-card :color="$store.state.vularApp.content.color">
          <v-list class="pt-0 vular-filter-list" :color="$store.state.vularApp.content.card.color">
            <template 
              v-for="(filter, index) in popFilters"
              
            >
              <v-subheader 
                :style="{background: $store.state.vularApp.content.color}"
                :key="index"
              >
              {{filter.title}}
              </v-subheader>
              <v-radio-group class="px-3" 
                v-model="inputValue.formModel[filter.field]"
                :key="index + 'radio-group'"
              >
                <v-radio
                  :label="filter.blankTitle"
                  :value="filter.blankValue"
                ></v-radio>
                <v-radio
                  v-for="(label, value) in filter.rules"
                  :key="value"
                  :label="label"
                  :value="value"
                ></v-radio>
              </v-radio-group>
            </template>
          </v-list>

          <v-card-actions>
            <v-spacer></v-spacer>

            <v-btn text @click="isPopedFilters = false">{{$t('base.close')}}</v-btn>
            <v-btn color="primary" text @click="onClear">{{$t('base.clear')}}</v-btn>
          </v-card-actions>
        </v-card>
      </v-menu>
    </template>
    <template v-else>
      <v-btn text rounded color="primary" class="mx-3"
        v-for="(action, index) in notPopedBatchActions"
        :key = "index"
        :small="isStick"
        @click="onBatchAction(action)"
        >
        {{action.title}}
        <v-icon right dark v-text="action.icon"></v-icon>
      </v-btn>
      <v-menu offset-y v-if="popBatchActions.length > 0">
        <template v-slot:activator="{ on }">
          <v-btn fab elevation="0" :color="$store.state.vularApp.content.card.color"
            :small="!isStick"
            :x-small="isStick"
            v-on="on"
         >
            <v-icon color="primary" class="top-small-button">mdi-dots-horizontal</v-icon>
          </v-btn>
        </template>
        <v-list :color="$store.state.vularApp.content.card.color" class="px-2">
          <v-list-item link
            v-for="(action, index) in popBatchActions"
            :key = "index"
            @click="onBatchAction(action)"
          >
            <v-list-item-icon>
              <v-icon color="primary" v-text="action.icon"></v-icon>
            </v-list-item-icon>
            <v-list-item-content>
              <v-list-item-title>{{action.title}}</v-list-item-title>
            </v-list-item-content>
          </v-list-item>
        </v-list>
      </v-menu>
    </template>
  </div>

</template>

<script>
  export default {
    name: 'vular-list-actions',
    props: {
      batchActions: { default: ()=>{return []} },
      filters : { default:()=>{return []} },
      isStick: {default : false},
      value: {default: ()=>{return []}},
      queryAction:{default:null},
    },

    data () {
      return {
        searchBoxFocused: false,
        isPopedFilters : false,
        keywords:"",
      }
    },

    computed:{
      inputValue: {
        get:function() {
          return this.value;
        },
        set:function(val) {
          this.$emit('input', val);
        },
      },

      shortcutFilters(){
        let filters = []
        this.filters.forEach(filter=>{
          if(filter.shortcut){
            filters.push(filter)
          }
        })
        return filters
      },

      notPopedBatchActions(){
        let actions = []
        this.batchActions.forEach(action=>{
          if(action.shortcut && !this.collopsed){
            actions.push(action)
          }
        })
        return actions
      },

      popBatchActions(){
        let actions = []
        this.batchActions.forEach(action=>{
          if(!action.shortcut || this.collopsed){
            actions.push(action)
          }
        })

        return actions
      },

      popFilters(){
        let pops = []
        this.filters.forEach(aFilter =>{
          if(!aFilter.shortcut || this.collopsed){
            pops.push(aFilter)
          }
        })

        return pops
      },

      selectedSome: {
        get:function() {
          return this.selectedCounts != 0;
        },
        set:function() {
        },
      },

      collopsed(){
        return this.$vuetify.breakpoint.xs || this.$vuetify.breakpoint.sm
      },

      searchboxWidth(){
        let width = 120
        if(this.searchBoxFocused){
          width = this.$vuetify.breakpoint.xs ? 150 : 200
        }
        return width
      },

      selectedCounts(){
        let counts = 0
        this.inputValue.rows.forEach(row=>{
          if(row.selected){
            counts ++
          }
        })

        return counts
      },

      selectedRows(){
        let rows = []
        this.inputValue.rows.forEach(row=>{
          if(row.selected){
            rows.push(row)
          }
        })

        return rows
      },

      popFilersSelected(){
        for(var i = 0; i < this.filters.length; i++){
          let filter = this.filters[i]
          if(filter.blankValue != this.inputValue.formModel[filter.field] && (!filter.shortcut || this.collopsed)){
            return true
          }
        }
        return false
      }
    },

    mounted(){
      this.keywords = this.inputValue.formModel.keywords
    },

    methods: {
      onSearch(){
        this.inputValue.formModel.keywords = this.keywords.trim()
      },

      onClearKeywords(){
        this.inputValue.formModel.keywords = ""
      },
      onFilter(filter, model){
        //this.$set(filter, 'model', model)
        this.inputValue.formModel[filter.field] = model
      },

      onClear(){
        this.isPopedFilters = false
         for(var i = 0; i < this.filters.length; i++){
          let filter = this.filters[i]
          if(!filter.shortcut || this.collopsed){
            this.inputValue.formModel[filter.field] = filter.blankValue
          }
        }
      },

      onSelectedChange(value){
        this.$emit('selectAll', value)
      },

      onBatchAction(actionCtrl){
        window.$bus.$emit("VularAction", actionCtrl.action, this.vularId, this.selectedRows)
      },
    },
  }
</script>

<style>
  .select-button{
    max-width: 200px;
    overflow: hidden;
    text-overflow:ellipsis;
    white-space: nowrap;
  }

  .list-search-box{
    margin-right: 50px; transition:all 0.3s;
  }

  .vular-filter-list{
    max-height: 400px;
    overflow-y: auto;
  }

  .filter-button{
    opacity: 0.6;
  }
</style>