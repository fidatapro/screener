<template>
  <v-data-table
    :headers="headers"
    :items="desserts"
    :rows-per-page-items="listSize" 
    :page="2"
    sort-by="calories"
    class="elevation-1"
    :loading = "loading"
    loading-text="Loading... Please wait"
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
        // { text: 'Symbol', value: 'coin_symbol' },
        { text: 'Price', value: 'coin_price' },
        { text: '30D%', value: 'percent_change_30d' },
        { text: 'YTD%', value: 'percent_change_ytd' },
        { text: '1Y%', value: 'percent_change_1year' },
      ],
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
        myHeaders.append("QC-Access-Key", "RIP1E1LIO3QOJKEH4QEB");
        myHeaders.append("QC-Secret-Key", "whRZJ8kT05LD1393oCm8jMs4269AoGrxEsbx1zCi0JBvK2gU");

        let requestOptions = {
          method: 'GET',
          headers: myHeaders,
          redirect: 'follow'
        };
        let coinData = [];
        let priceData = [];
        let percentData = [];

        await fetch("https://quantifycrypto.com/api/v1/coins/prices?rank_from=1&rank_to=1000", requestOptions)
        .then((response) => response.json())
        .then((result) => {
          priceData = result.data;
        })
        .catch(error => console.log('error', error));

        await fetch("https://quantifycrypto.com/api/v1/coins/percent-change?rank_from=1&rank_to=1000", requestOptions)
        .then((response) => response.json())
        .then((result) => {
          percentData = result.data;
          console.log(percentData);
        })
        .catch(error => console.log('error', error));

        await fetch("https://quantifycrypto.com/api/v1/coins/list", requestOptions)
        .then((response) => response.json())
        .then((result) => {
          coinData = result.data;
        })
        .catch(error => console.log('error', error));
        if(coinData && priceData && percentData) {

          this.desserts = coinData.map(async (item) => {
            const price = priceData.filter((price_item) => price_item.id == item.id);
            const percent = percentData.filter((percent_item) => percent_item.id == item.id);

            item.status = false;

            if(price && price.length > 0) {
              item.coin_price = price[0].coin_price;
            }
            else {
              item.coin_price = "";
            }

            if(percent && percent.length > 0) {
              item.percent_change_5min = percent[0].percent_change_5min;
              item.percent_change_15min = percent[0].percent_change_15min;
              item.percent_change_1h = percent[0].percent_change_1h;
              item.percent_change_2h = percent[0].percent_change_2h;
              item.percent_change_4h = percent[0].percent_change_4h;
              item.percent_change_24h = percent[0].percent_change_24h;
              item.percent_change_7d = percent[0].percent_change_7d;
              item.percent_change_14d = percent[0].percent_change_14d;
              item.percent_change_30d = percent[0].percent_change_30d;
              item.percent_change_ytd = percent[0].percent_change_ytd;
              item.percent_change_1year = percent[0].percent_change_1year;
            }
            else {
              item.percent_change_5min = "";
              item.percent_change_15min = "";
              item.percent_change_1h = "";
              item.percent_change_2h = "";
              item.percent_change_4h = "";
              item.percent_change_24h = "";
              item.percent_change_7d = "";
              item.percent_change_14d = "";
              item.percent_change_30d = "";
              item.percent_change_ytd = "";
              item.percent_change_1year = "";
            }
            return item;
          });
        }
        this.desserts = coinData;
        console.log('==>', this.desserts);
        this.loading = false;
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