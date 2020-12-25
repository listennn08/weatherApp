<template>
  <div class="container">
    <ul class="nav nav-tabs">
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
        class="table"
        :key="transition"
      >
        <thead>
          <tr>
            <th>縣市</th>
            <th>時間</th>
            <th
              v-for="t in time"
              :key="t"
            >
              {{ t }}
            </th>
          </tr>
        </thead>
        <tbody>
          <template
            v-for="(d, i) in showData"
            :key="i"
          >
            <tr class="light align-middle">
              <td
                rowspan="2"
              >
                {{ d.locationName }}
              </td>
              <td>白天</td>
              <td
                v-for="t in time"
                :key="t"
              >
                {{ getInterval(t, d.PoP12h.time, 'm')?.elementValue?.[0].value.trim() || 0 }}%
                <br>
                <span v-if="getInterval(t, d.MinT.time, 'm')?.elementValue?.[0].value">
                  {{ getInterval(t, d.MinT.time, 'm')?.elementValue?.[0].value }}°C -
                  {{ getInterval(t, d.MaxT.time, 'm')?.elementValue?.[0].value }}°C
                </span>
                <span v-else></span>
              </td>
            </tr>
            <tr class="night align-middle">
              <td>晚上</td>
              <td
                v-for="t in time"
                :key="t"
              >
                {{ getInterval(t, d.PoP12h.time, 'n')?.elementValue?.[0].value.trim() || 0 }}%
                <br>
                <span v-if="getInterval(t, d.MinT.time, 'n')?.elementValue?.[0].value">
                  {{ getInterval(t, d.MinT.time, 'n')?.elementValue?.[0].value}}°C -
                  {{ getInterval(t, d.MaxT.time, 'n')?.elementValue?.[0].value}}°C
                </span>
                <span v-else></span>
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

type Areas = {
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
  locationName: string;
  PoP12h: TWeatherElement;
  MinT: TWeatherElement;
  MaxT: TWeatherElement;
}

type TWeatherData = {
  startTime: string;
  endTime: string;
  elementValue: [
    {
      value: string;
      measures: string;
    }
  ];
}

const auth = 'CWB-B064CB6C-3660-4D60-A4B8-0988834FD02E';
const url = `https://opendata.cwb.gov.tw/api/v1/rest/datastore/F-D0047-091?Authorization=${auth}&elementName=MinT,MaxT,PoP12h`;

const areas: Areas = {
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
    const { data } = await axios.get(url);
    const time = computed<string[]>(() => {
      const { location } = data.records.locations[0];
      return Array.from(
        new Set(location[0].weatherElement[0].time.map((el: TWeatherData) => format(new Date(el.startTime), 'yyyy-MM-dd'))),
      );
    });

    const weatherData: TWeatherItem[] = data.records.locations[0].location.map((el: TJSONData) => {
      const [PoP12h, MinT, MaxT] = el.weatherElement;
      return {
        lat: el.lat,
        locationName: el.locationName,
        PoP12h,
        MinT,
        MaxT,
      };
    }).sort((a: TJSONData, b: TJSONData) => parseFloat(b.lat) - parseFloat(a.lat));
    const transition = ref(false);

    const currentTab = reactive({
      value: '全部',
      isCurrentTab: computed(() => (tab: string) => (tab === currentTab.value)),
      changeTab: (tab: string) => {
        currentTab.value = tab;
        transition.value = !transition.value;
      },
    });
    const showData = computed(() => weatherData.filter((el) => {
      if (currentTab.value === '全部') return el;
      return areas[currentTab.value].includeCity.includes(el.locationName);
    }));

    const getInterval = computed(() => (
      currentDate: string,
      item: TWeatherData[],
      section: string,
    ) => item.filter((el) => {
      const pivot = new Date(`${currentDate} 12:00:00`).getTime();
      const t = new Date(el.startTime).getTime();
      return format(new Date(el.startTime), 'yyyy-MM-dd') === currentDate
        && (section === 'm'
          ? t - pivot < 0
          : t - pivot > 0);
    })[0]);

    return {
      showData,
      areas,
      currentTab,
      transition,
      time,
      getInterval,
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
