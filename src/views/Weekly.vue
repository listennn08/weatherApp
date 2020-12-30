<template>
  <div class="container">
    <div class="w-50 d-flex justify-content-end float-end">
      <div
        class="me-3"
      >
        溫度單位：
        <a
          href="#"
          @click.prevent="toggleTempartureType"
          class="text-decoration-none"
          title="點我更換溫度單位"
        >
          {{ tempartureType ? '°F' : '°C'}}
        </a>
      </div>
      <div>
        有效時間：{{ dateDuration[0] }} ~ {{ dateDuration[1] }}
      </div>
    </div>
    <div class="clearfix"></div>
    <ul class="nav nav-tabs mb-3">
      <li
        v-for="(_, area) in areas"
        :key="area"
        class="nav-item"
      >
        <a
          class="nav-link"
          href="#"
          @click.prevent="currentTab.changeTab(area)"
          :class="{'active': currentTab.isCurrentTab(area)}"
        >
          {{ area }}
        </a>
      </li>
    </ul>

    <transition
      name='list'
      tag='table'
    >
      <table
        class="table table-bordered"
        :key="transition"
      >
        <thead>
          <tr class="align-middle">
            <th class="bg-primary text-white">縣市</th>
            <th style="width: 5%" class="bg-primary text-white">時間</th>
            <th
              v-for="t in time.value"
              :key="t"
              :class="time.isHoliday(t) ? 'bg-danger' : 'bg-primary'"
              class="text-white"
            >
              {{ time.displayTime(t) }}
              <br>
              {{ time.getWeekDay(t) }}
            </th>
          </tr>
        </thead>
        <tbody>
          <template
            v-for="(item, location) in showData"
            :key="location"
          >
            <tr class="light align-middle">
              <td
                rowspan="2"
                class="bg-info text-white"
              >
                {{ location }}
              </td>
              <td
                class="bg-primary text-white"
              >
                白天
              </td>
              <td
                v-for="d in item"
                :key="location+d"
              >
                {{ d.PoP12h[0] || 0 }}%
                <br>
                {{ tempartureConverter(d.MinT[0]) }} - {{ tempartureConverter(d.MaxT[0]) }}
              </td>
            </tr>
            <tr class="night align-middle">
              <td
                class="bg-primary text-white"
              >
                晚上
              </td>
              <td
                v-for="d in item"
                :key="location+d"
              >
                {{ d.PoP12h[1] || 0 }}%
                <br>
                {{ tempartureConverter(d.MinT[1]) }} - {{ tempartureConverter(d.MaxT[1]) }}
              </td>
            </tr>
          </template>
        </tbody>
      </table>
    </transition>
  </div>
</template>

<script lang="ts">
import {
  ref, reactive, defineComponent, computed,
} from 'vue';
import axios from 'axios';
import { format } from 'date-fns';

enum WeekDay {
  '星期日',
  '星期一',
  '星期二',
  '星期三',
  '星期四',
  '星期五',
  '星期六',
}

type TAreas = {
  [name: string]: {
    includeCity: string[];
  };
}

type TJSONData = {
  geocode: string;
  lat: string;
  locationName: string;
  lon: string;
  weatherElement: TWeatherElement[];
}

type TWeatherElement = {
  lat: string;
  elementName: string;
  description: string;
  time: [
    {
      startTime: string;
      endTime: string;
      elementValue: [
        {
          value: string;
          measures: string;
        }
      ];
    }
  ];
}

type TWeatherItem = {
  startTime: string;
  endTime: string;
  elementValue: [
    {
      value: string;
      measures: string;
    }
  ];
}

type TWeatherData = {
  [location: string]: {
    [time: string]: {
      [elementName: string]: string[];
    };
  };
};

const auth = process.env.VUE_APP_SECRECT_KEY;
const url = `https://opendata.cwb.gov.tw/api/v1/rest/datastore/F-D0047-091?Authorization=${auth}&elementName=MinT,MaxT,PoP12h`;

const areas: TAreas = {
  全部: {
    includeCity: [],
  },
  北部: {
    includeCity: ['臺北市', '新北市', '基隆市', '桃園市', '新竹市', '新竹縣'],
  },
  中部: {
    includeCity: ['苗栗縣', '臺中市', '彰化縣', '南投縣', '雲林縣', '嘉義市', '嘉義縣'],
  },
  南部: {
    includeCity: ['臺南市', '高雄市', '屏東縣'],
  },
  東部: {
    includeCity: ['宜蘭縣', '花蓮縣', '臺東縣'],
  },
  外島: {
    includeCity: ['澎湖縣', '金門縣', '連江縣'],
  },
};

export default defineComponent({
  async setup() {
    const resp = await axios.get(url);
    const { location: locations } = resp.data.records.locations[0];
    const weatherData: TWeatherData = {};
    locations.forEach((location: TJSONData) => {
      location.weatherElement.forEach((el: TWeatherElement) => {
        el.time.forEach((time) => {
          if (!weatherData[location.locationName]) {
            weatherData[location.locationName] = {};
          }
          const date = Intl.DateTimeFormat('zh-tw').format(new Date(time.startTime));
          if (!weatherData[location.locationName][date]) {
            weatherData[location.locationName][date] = {};
          }
          if (!weatherData[location.locationName][date][el.elementName]) {
            weatherData[location.locationName][date] = {
              ...weatherData[location.locationName][date],
              [el.elementName]: [],
            };
          }
          weatherData[location.locationName][date][el.elementName].push(
            time.elementValue[0].value.trim(),
          );
        });
      });
    });

    const time = reactive({
      value: (() => Array.from(new Set<string>(locations[0]
        .weatherElement[0]
        .time.map((el: TWeatherItem) => {
          const dt = new Date(el.startTime);
          return [dt.getFullYear(), dt.getMonth(), dt.getDate()];
        })
        .map(JSON.stringify)))
        .map((el) => JSON.parse(el)))(),
      displayTime: computed(() => (t: number[]) => format(new Date(t[0], t[1], t[2]), 'MM/dd')),
      getWeekDay: (t: number[]) => WeekDay[new Date(t[0], t[1], t[2]).getDay()],
      isHoliday: computed(() => (dt: number[]) => {
        const [yyyy, mm, dd] = dt;
        const d = new Date(yyyy, mm, dd).getDay();
        return d === 0 || d === 6;
      }),
    });

    const dateDuration = (() => {
      const [sY, sM, sD] = time.value[0];
      const [eY, eM, eD] = time.value[time.value.length - 1];
      return [
        format(new Date(sY, sM, sD, 12, 0), 'MM/dd HH:ss'),
        format(new Date(eY, eM, eD + 1, 6, 0), 'MM/dd HH:ss'),
      ];
    })();
    // const weatherData: TWeatherItem[] = data.records.locations[0].location
    //   .map((el: TJSONData) => {
    //     const [PoP12h, MinT, MaxT] = el.weatherElement;
    //     return {
    //       lat: el.lat,
    //       locationName: el.locationName,
    //       PoP12h,
    //       MinT,
    //       MaxT,
    //     };
    //   })
    //   .sort((a: TJSONData, b: TJSONData) => parseFloat(b.lat) - parseFloat(a.lat));
    const transition = ref(false);

    const currentTab = reactive({
      value: '全部',
      isCurrentTab: computed(() => (tab: string) => (tab === currentTab.value)),
      changeTab: (tab: string) => {
        if (currentTab.value !== tab) {
          currentTab.value = tab;
          transition.value = !transition.value;
        }
      },
    });
    const showData = computed(() => Object.keys(weatherData).reduce((acc, key) => {
      if (currentTab.value === '全部') return { ...acc, [key]: weatherData[key] };
      if (areas[currentTab.value].includeCity.includes(key)) {
        return { ...acc, [key]: weatherData[key] };
      }
      return { ...acc };
    }, {}));

    const tempartureType = ref(false);
    const toggleTempartureType = () => { tempartureType.value = !tempartureType.value; };
    const tempartureConverter = (temp: number): string => {
      if (temp || temp === 0) {
        if (tempartureType.value) {
          return `${(temp * (9 / 5) + 32).toFixed(0)}°F`;
        }
        return `${temp}°C`;
      }
      return '';
    };

    return {
      showData,
      areas,
      currentTab,
      transition,
      time,
      dateDuration,
      tempartureType,
      toggleTempartureType,
      tempartureConverter,
    };
  },
});
</script>

<style lang="scss" scoped>
.list-enter-active,
.list-leave-active {
  transition: all .5s ease-out;
}

.list-enter-from,
.list-leave-to {
  opacity: 0;
  transform: translateY(30px);
}

.list-enter-to {
  transition: all .3s .5s ease-out;
  opacity: .5;
  transform: translateY(30);
}
</style>
