<template>
  <v-data-table
    :headers="headers"
    :items="desserts"
    :items-per-page="10"
    :page="2"
    sort-by="calories"
    class="elevation-1"
    :options.sync="pagination"
    @update:page="updatePage()"
    :footer-props="footerProps"
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
        ${{ item.coin_price }}
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
    <!-- eslint-disable-next-line -->
    <template v-slot:item.rsi_2h="{item}">
      <span
        :class="getRsiColor(item.rsi_2h)"
      >
        {{ item.rsi_2h }}
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
        { text: 'Name', width: '200px', value: 'coin_name' },
        { text: 'Price', value: 'coin_price', align: 'end' },
        { text: '30D%', value: 'percent_change_30d' },
        { text: 'YTD%', value: 'percent_change_ytd' },
        { text: '1Y%', value: 'percent_change_1year' },
        { text: 'RSI 2H', value: 'rsi_2h' },
        { text: 'MACD 2H', value: 'macd_hist_2h' },
        { text: 'ATR 2H', value: 'atr_2h' },
        { text: 'BB 2H', value: 'bollinger_bands_lower_2h' },
        { text: 'MA 12H', value: 'ma_1h_12' },
        { text: 'EMA 12H', value: 'ema_1h_12' },
      ],
      pagination: {},
      listSize: [10, 25, 50, 100],
      desserts: [],
      connection: null,
      loading: true,
      footerProps: {
        showFirstLastPage: true,
        // disableItemsPerPage: true,
        showCurrentPage: true,
        itemsPerPageOptions: [10, 50, 100, -1],
      }
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

        await fetch("https://quantifycrypto.com/api/v1/coins/list", requestOptions)
        .then((response) => response.json())
        .then((result) => {
          coinData = result.data;
        })
        .catch(error => console.log('error', error));
        if(coinData) {
         
          for(let i =  0, cnt = 0; cnt < 500; i++)
          {
            var formdata = new FormData();
            formdata.append("symbol", coinData[i].coin_symbol);

            var detailRequest = {
              method: 'POST',
              body: formdata,
              redirect: 'follow'
            };
            await fetch("https://pro.coingen.net/api/quantifycrypto-coin", detailRequest)
            .then((response) => response.json())
            .then((result) => {
              if(result) {
                coinData[i].coin_price = this.getRealValue(result.coin_price);
                coinData[i].percent_change_30d = this.getRealValue(result.percent_change_30d);
                coinData[i].percent_change_ytd = this.getRealValue(result.percent_change_ytd);
                coinData[i].percent_change_1year = this.getRealValue(result.percent_change_1year);
                coinData[i].rsi_2h = result.rsi_2h;
                coinData[i].macd_hist_2h = result.macd_hist_2h;
                coinData[i].atr_2h = result.atr_2h;
                coinData[i].bollinger_bands_lower_2h = result.bollinger_bands_lower_2h;
                coinData[i].ma_1h_12 = result.ma_1h_12;
                coinData[i].ema_1h_12 = result.ema_1h_12;
              }
            })
            .catch(error => console.log(i, ':error', error));
            this.desserts.push(coinData[i]);
            cnt++;
            // console.log(i);
          }

        }
        // this.desserts = coinData;
        this.loading = false;
        console.log('==>', this.desserts);
      },
      getRealValue ( value ) {
        if(Math.abs(value) > 1) {
          return Math.round(value * 100) / 100;
        } 
        else if(value == 0) {
          return 0;
        }
        else {
          const pos = Math.floor(Math.log10(Math.abs(value))) * -1 + 1;
          return Math.floor(value * Math.pow(10, pos)) / Math.pow(10, pos);
        }
      },
      async onSocketMessage ( event ) {
        if(this.desserts && event.data) {
          const prices = JSON.parse(event.data);
          let temp = this.desserts;
          // console.log(prices);
          for(let i = 0; i < temp.length; i++)
          {
            if(temp[i] &&  prices[temp[i].coin_name.toLowerCase()]) {
              temp[i].coin_price = this.getRealValue(prices[temp[i].coin_name.toLowerCase()]);
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
      getRsiColor ( color_val) {
        if(Number(color_val) > 70) {
          return 'color-green';
        }
        else if(Number(color_val) < 30) {
          return 'color-red';
        }
        else {
          return 'color-orange'
        }
      },
      getSrc ( src ) {
        return "https://quantifycrypto.s3-us-west-2.amazonaws.com/pictures/crypto-img/32/icon/" + src.toLowerCase() + ".png";
      },
      async updatePage ( ) {
        if(this.loading) return;
        this.loading = true;
        
        for(let i = (this.pagination.page - 1) * 10; i < Math.min(this.pagination.page * 10 , this.desserts.length); i++)
        {
          let formdata = new FormData();
          formdata.append("symbol", this.desserts[i].coin_symbol);

          let detailRequest = {
            method: 'POST',
            body: formdata,
            redirect: 'follow'
          };
          if(this.desserts[i].percent_change_30d == undefined) {
            await fetch("https://pro.coingen.net/api/quantifycrypto-coin", detailRequest)
            .then((response) => response.json())
            .then((result) => {
              if(result) {
                this.desserts[i].coin_price = this.getRealValue(result.coin_price);
                this.desserts[i].percent_change_30d = this.getRealValue(result.percent_change_30d);
                this.desserts[i].percent_change_ytd = this.getRealValue(result.percent_change_ytd);
                this.desserts[i].percent_change_1year = this.getRealValue(result.percent_change_1year);
                this.desserts[i].rsi_2h = result.rsi_2h;
                this.desserts[i].macd_hist_2h = result.macd_hist_2h;
                this.desserts[i].atr_2h = result.atr_2h;
                this.desserts[i].bollinger_bands_lower_2h = result.bollinger_bands_lower_2h;
                this.desserts[i].ma_1h_12 = result.ma_1h_12;
                this.desserts[i].ema_1h_12 = result.ema_1h_12;
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
  .color-orange {
    color: #ff9800;
  }
</style>