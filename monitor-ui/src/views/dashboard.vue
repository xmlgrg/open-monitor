<template>
  <div>
    <section v-if="isPlugin">
      <div style="margin: 100px 0;text-align: center;font-size:14px">
        {{$t('tips.dashboardEmpty')}}
      </div>
    </section>
    <section v-else>
      <header>
        <div class="search-zone">
          <span class="params-title">{{$t('field.relativeTime')}}：</span>
          <Select v-model="viewCondition.timeTnterval" :disabled="disableTime" style="width:80px"  @on-change="initPanals">
            <Option v-for="item in dataPick" :value="item.value" :key="item.value">{{ item.label }}</Option>
          </Select>
        </div>
        <div class="search-zone">
          <span class="params-title">{{$t('placeholder.refresh')}}：</span>
          <Select v-model="viewCondition.autoRefresh" :disabled="disableTime" style="width:100px" @on-change="initPanals" :placeholder="$t('placeholder.refresh')">
            <Option v-for="item in autoRefreshConfig" :value="item.value" :key="item.value">{{ item.label }}</Option>
          </Select>
        </div>
        <div class="search-zone">
          <span class="params-title">{{$t('field.timeInterval')}}：</span>
          <DatePicker 
            type="datetimerange" 
            :value="viewCondition.dateRange" 
            format="yyyy-MM-dd HH:mm:ss" 
            placement="bottom-start" 
            @on-change="datePick" 
            :placeholder="$t('placeholder.datePicker')" 
            style="width: 320px">
          </DatePicker>
        </div>
      </header>
      <div>
        <grid-layout
          :layout.sync="layoutData"
          :col-num="12"
          :row-height="30"
          :is-draggable="false"
          :is-resizable="false"
          :is-mirrored="false"
          :vertical-compact="true"
          :use-css-transforms="true"
          >
          <grid-item v-for="(item,index) in layoutData"
            class="c-dark"
            :x="item.x"
            :y="item.y"
            :w="item.w"
            :h="item.h"
            :i="item.i"
            :key="index"
            @resized="resizeEvent">   
            <div class="c-dark" style="display:flex;justify-content:flex-end;padding:0 32px;">
              <div class="header-grid header-grid-name">
                {{item.i}}
              </div>
            </div>
            <section>
              <div v-for="(chartInfo,chartIndex) in item._activeCharts" :key="chartIndex">
                <CustomChart v-if="['line','bar'].includes(chartInfo.chartType)" :chartInfo="chartInfo" :chartIndex="index" :params="viewCondition"></CustomChart>
                <CustomPieChart v-if="chartInfo.chartType === 'pie'" :chartInfo="chartInfo" :chartIndex="index" :params="viewCondition"></CustomPieChart>
              </div>
            </section>
          </grid-item>
        </grid-layout>
      </div>
    </section>
  </div>
</template>

<script>
import VueGridLayout from 'vue-grid-layout'
import {dataPick, autoRefreshConfig} from '@/assets/config/common-config'
import {resizeEvent} from '@/assets/js/gridUtils.ts'
import CustomChart from '@/components/custom-chart'
import CustomPieChart from '@/components/custom-pie-chart'
export default {
  name: '',
  data() {
    return {
      isPlugin: false,
      viewCondition: {
        timeTnterval: -1800,
        dateRange: ['', ''],
        autoRefresh: 10,
      },
      disableTime: false,
      dataPick: dataPick,
      autoRefreshConfig: autoRefreshConfig,
      viewData: [],
      layoutData: [
        //   {'x':0,'y':0,'w':2,'h':2,'i':'0'},
        //   {'x':1,'y':1,'w':2,'h':2,'i':'1'},
      ]
    }
  },
  mounted() {
    this.getDashboardData()
  },
  methods: {
    datePick (data) {
      this.viewCondition.dateRange = data
      this.disableTime = false
      if (this.viewCondition.dateRange[0] && this.viewCondition.dateRange[1]) {
        if (this.viewCondition.dateRange[0] === this.viewCondition.dateRange[1]) {
          this.viewCondition.dateRange[1] = this.viewCondition.dateRange[1].replace('00:00:00', '23:59:59')
        }
        this.disableTime = true
        this.viewCondition.autoRefresh = 0
      }
      this.initPanals()
    },
    getDashboardData () {
      this.$root.$httpRequestEntrance.httpRequestEntrance('GET',this.$root.apiCenter.template.get, '', responseData => {
        if (responseData.cfg === '' || responseData.cfg === '[]') {
          if (window.request) {
            this.isPlugin = true
          } else {
            this.$router.push({path: 'portal'})
          }
        }else {
          this.viewData = JSON.parse(responseData.cfg) 
          this.initPanals()
        }
      })
    },
    initPanals () {
      let tmp = []
      this.viewData.forEach((item) => {
        let params = []
        item.query.forEach( _ => {
          params.push({
            endpoint: _.endpoint,
            metric: _.metricLabel,
            prom_ql: _.metric,
          })
        })
        let height = (item.viewConfig.h) * 30
        let _activeCharts = []
        _activeCharts.push({
          style: `height:${height}px;`,
          panalUnit: item.panalUnit,
          elId: item.viewConfig.id,
          chartParams: params,
          chartType: item.chartType                                                     
        })
        item.viewConfig._activeCharts = _activeCharts
        tmp.push(item.viewConfig)
      })
      this.layoutData = tmp
    },
    resizeEvent: function(i, newH, newW, newHPx, newWPx){
      resizeEvent(this, i, newH, newW, newHPx, newWPx)
    }
  },
  components: {
    GridLayout: VueGridLayout.GridLayout,
    GridItem: VueGridLayout.GridItem,
    CustomChart,
    CustomPieChart
  },
}
</script>

<style scoped lang="less">
header {
  margin: 16px 8px;
}
.search-zone {
  display: inline-block;
}
.params-title {
  margin-left: 4px;
  font-size: 13px;
}

.header-grid {
  flex-grow: 1;
  text-align: center;
  line-height: 32px;
  i {
    margin: 0 4px;
    cursor: pointer;
  } 
}
.vue-grid-item {
  border-radius: 4px;
}
.vue-grid-item:not(.vue-grid-placeholder) {
    background: @gray-f;
    border: 1px solid @gray-f;
}
</style>
