<template>
  <v-data-table
    :headers="headers"
    :items="desserts"
    :page="2"
    sort-by="calories"
    class="elevation-1"
    :loading = "loading"
    loading-text="Loading... Please wait"
    :options.sync="pagination"
    @update:page="updatePage()"
  >
    <template v-slot:top>
      <v-toolbar
        flat
      >
        <v-toolbar-title>Cryptocurrency Screener</v-toolbar-title>
        <v-divider
          class="mx-4"
          inset
          vertical
        ></v-divider>
        <v-spacer></v-spacer>
      </v-toolbar>
    </template>
    <!-- eslint-disable-next-line -->
    <template v-slot:no-data>
      <v-btn
        color="primary"
        @click="initialize"
      >
        Reset
      </v-btn>
    </template>
    <!-- eslint-disable-next-line -->
    <template v-slot:item.coin_name="{item}">
      <img style="width: 30px; padding-right: 8px; vertical-align: middle" :src="getSrc(item.coin_symbol)" />
      {{ item.coin_name }}
    </template>
    <!-- eslint-disable-next-line -->
    <template v-slot:item.coin_price="{item}">
      <div
        :class="getStatus(item.status, item.id)"
      >
        {{ item.coin_price }}
      </div>
    </template>
    <!-- eslint-disable-next-line -->
    <template v-slot:item.percent_change_30d="{item}">
      <span
        :class="getColor(item.percent_change_30d)"
      >
        {{ item.percent_change_30d }} %
      </span>
    </template>
    <!-- eslint-disable-next-line -->
    <template v-slot:item.percent_change_ytd="{item}">
      <span
        :class="getColor(item.percent_change_ytd)"
      >
        {{ item.percent_change_ytd }} %
      </span>
    </template>
    <!-- eslint-disable-next-line -->
    <template v-slot:item.percent_change_1year="{item}">
      <span
        :class="getColor(item.percent_change_1year)"
      >
        {{ item.percent_change_1year }} %
      </span>
    </template>
  </v-data-table>
</template>

<script>
  export default {
    data: () => ({
      headers: [
        {
          text: '#',
          align: 'start',
          sortable: false,
          value: 'rank',
        },
        { text: 'Name', value: 'coin_name' },
        { text: 'Price', value: 'coin_price' },
        { text: '30D%', value: 'percent_change_30d' },
        { text: 'YTD%', value: 'percent_change_ytd' },
        { text: '1Y%', value: 'percent_change_1year' },
        { text: 'RSI 2H', value: 'rsi_2h' },
        { text: 'MACD 2H', value: 'macd_hist_2h' },
        { text: 'ATR 2H', value: 'atr_2h' },
        { text: 'BB 2H', value: 'bollinger_bands_lower_2h' },
        { text: 'TREND CRITICAL', value: 'trend_critical' },
        { text: 'TREND DAILY', value: 'trend_daily' },
        { text: 'SUPPORT', value: 'support' },
        { text: 'RESISTANCE', value: 'resistance' },
    ],
      pagination: {},
      listSize: [10, 25, 50, 100],
      desserts: [],
      connection: null,
      loading: true,
    }),

    watch: {
      dialog (val) {
        val || this.close()
      },
      dialogDelete (val) {
        val || this.closeDelete()
      },
    },

    created () {
      this.initialize();
    },

    mounted () {
      this.connection = new WebSocket("wss://ws.coincap.io/prices?assets=ALL");
      this.connection.onmessage = this.onSocketMessage;
      // console.log(this.connection);
    },

    methods: {
      async initialize () {
        let myHeaders = new Headers();
        myHeaders.append("QC-Access-Key", "EQF6H18BNERJH2ML84F1");
        myHeaders.append("QC-Secret-Key", "HzFyQix9rmtKp9ti9GAfLs7twqd1g5RAmqls0jJz2Z8bNQBz");

        let requestOptions = {
          method: 'GET',
          headers: myHeaders,
          redirect: 'follow'
        };
        let coinData = [];
        // let priceData = [];
        // let percentData = [];

        // await fetch("https://quantifycrypto.com/api/v1/coins/prices?rank_from=1&rank_to=1000", requestOptions)
        // .then((response) => response.json())
        // .then((result) => {
        //   priceData = result.data;
        // })
        // .catch(error => console.log('error', error));

        // await fetch("https://quantifycrypto.com/api/v1/coins/percent-change?rank_from=1&rank_to=1000", requestOptions)
        // .then((response) => response.json())
        // .then((result) => {
        //   percentData = result.data;
        //   console.log(percentData);
        // })
        // .catch(error => console.log('error', error));

        await fetch("https://quantifycrypto.com/api/v1/coins/list", requestOptions)
        .then((response) => response.json())
        .then((result) => {
          coinData = result.data;
        })
        .catch(error => console.log('error', error));
        if(coinData) {

          for(let i =  0; i < coinData.length; i++)
          {
            await fetch("https://quantifycrypto.com/api/v1/coins/" + coinData[i].id, requestOptions)
            .then((response) => response.json())
            .then((result) => {
              if(result.data) {
                coinData[i].coin_price = result.data.coin_price;
                coinData[i].percent_change_30d = result.data.percent_change_30d;
                coinData[i].percent_change_ytd = result.data.percent_change_ytd;
                coinData[i].percent_change_1year = result.data.percent_change_1year;
                coinData[i].rsi_2h = result.data.rsi_2h;
                coinData[i].macd_hist_2h = result.data.macd_hist_2h;
                coinData[i].atr_2h = result.data.atr_2h;
                coinData[i].bollinger_bands_lower_2h = result.data.bollinger_bands_lower_2h;
                coinData[i].trend_critical = '-';
                coinData[i].trend_daily = '-';
                coinData[i].support = '-';
                coinData[i].resistance = '-';
              }
            })
            .catch(error => console.log(i, ':error', error));
            this.desserts.push(coinData[i]);
            // console.log(i);
          }

        }
        // this.desserts = coinData;
        this.loading = false;
        console.log('==>', this.desserts);
      },
      onSocketMessage ( event ) {
        if(this.desserts && event.data) {
          const prices = JSON.parse(event.data);
          let temp = this.desserts;
          // console.log(prices);
          for(let i = 0; i < temp.length; i++)
          {
            if(temp[i] &&  prices[temp[i].coin_name.toLowerCase()]) {
              temp[i].coin_price = prices[temp[i].coin_name.toLowerCase()];
              temp[i].status = true;
            }
            else {
              temp[i].status = false;
            }
          }
          this.desserts = temp;
          // console.log(this.desserts[0]);
        }
      },
      getStatus ( status, id ) {
        const idx = this.desserts.findIndex((item) => item.id == id);
        if(idx > -1) {
          if(status == true){
            // this.desserts[idx].status = false;
            return 'changed';
          } 
        }
        return 'normal';
      },
      getColor ( color_val) {
        if(Number(color_val) >= 0) {
          return 'color-green';
        }
        else {
          return 'color-red';
        }
      },
      getSrc ( src ) {
        return "https://quantifycrypto.s3-us-west-2.amazonaws.com/pictures/crypto-img/32/icon/" + src.toLowerCase() + ".png";
      },
      async updatePage ( ) {
        if(this.loading) return;
        this.loading = true;
        let myHeaders = new Headers();
        myHeaders.append("QC-Access-Key", "EQF6H18BNERJH2ML84F1");
        myHeaders.append("QC-Secret-Key", "HzFyQix9rmtKp9ti9GAfLs7twqd1g5RAmqls0jJz2Z8bNQBz");

        let requestOptions = {
          method: 'GET',
          headers: myHeaders,
          redirect: 'follow'
        };
        
        for(let i = (this.pagination.page - 1) * 10; i < Math.min(this.pagination.page * 10 , this.desserts.length); i++)
        {
          if(!this.desserts[i].percent_change_30d) {
            await fetch("https://quantifycrypto.com/api/v1/coins/" + this.desserts[i].id, requestOptions)
            .then((response) => response.json())
            .then((result) => {
              if(result.data) {
                this.desserts[i].coin_price = result.data.coin_price;
                this.desserts[i].percent_change_30d = result.data.percent_change_30d;
                this.desserts[i].percent_change_ytd = result.data.percent_change_ytd;
                this.desserts[i].percent_change_1year = result.data.percent_change_1year;
                this.desserts[i].rsi_2h = result.data.rsi_2h;
                this.desserts[i].macd_hist_2h = result.data.macd_hist_2h;
                this.desserts[i].atr_2h = result.data.atr_2h;
                this.desserts[i].bollinger_bands_lower_2h = result.data.bollinger_bands_lower_2h;
                this.desserts[i].trend_critical = '-';
                this.desserts[i].trend_daily = '-';
                this.desserts[i].support = '-';
                this.desserts[i].resistance = '-';
              }
            })
            .catch(error => console.log('error', error));
          }
        }
        this.loading = false;
        // console.log(this.pagination);
      }
    },
  }
</script>
<style>
  @keyframes price-change {
    from {background-color: #4caf50;}
    to {background-color: black;}
  }

  .changed {
    animation-name: price-change;
    animation-duration: 2s;
  }

  .normal {
    color: white;
  }

  .color-red {
    color: #f44336;
  }
  .color-green {
    color: #4caf50;
  }
</style>