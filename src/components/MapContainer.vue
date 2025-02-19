<template>
  <div class="map-wrap">
    <input
      v-model="positionInput"
      id="searchInputId"
      class="search_input"
      placeholder="输入关键词搜索位置"
      style="height: 25px; width: 26%; margin-left: 1%"
    />
    <div id="container"></div>
  </div>
</template>
<script setup>
import { onMounted, onUnmounted, ref, watch } from 'vue';
import AMapLoader from '@amap/amap-jsapi-loader';
const positionInput = ref('');
let map = null;

const position = ref(null);
let marker = null; //点标记
const geocoder = ref(null); //地理编码器
const selectAddress = ref(''); //选点地理位置
//初始化地图
const initMap = async () => {
  try {
    const res = await getLocationH5();

    console.log('h5位置-----', res);
    setMap(res);
  } catch (error) {
    console.log('initMap-----', error);
    setMap({ longitude: '121.863795', latitude: '30.921259' });//[121.863795,30.921259] 滴水湖位置 [121.456609,31.182844]平安大厦
  }
};
const setMap = (res) => {
  AMapLoader.load({
    key: 'de8bef047ccfdfd99cd1b540b2f2d360', // 申请好的Web端开发者Key，首次调用 load 时必填
    version: '2.0', // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
    plugins: [
      'AMap.Scale',
      'AMap.AutoComplete',
      'AMap.PlaceSearch',
      'AMap.Marker',
      'AMap.Geolocation',
      'AMap.Geocoder',
    ], // 需要使用的的插件列表，如比例尺'AMap.Scale'等
  })
    .then((AMap) => {
      map = new AMap.Map('container', {
        // 设置地图容器id
        zoom: 15, // 初始化地图级别
        center: [res.longitude, res.latitude],
        viewMode: '2D', //设置地图模式
      });

      geocoder.value = new AMap.Geocoder(); //

      var autoOptions = {
        input: 'searchInputId', //searchInputId search_input
      };
      var auto = new AMap.AutoComplete(autoOptions);
      var placeSearch = new AMap.PlaceSearch({
        map: map,
      });
      //搜索位置
      auto.on('select', (e) => {
        placeSearch.setCity(e.poi.adcode); // 设置搜索范围为选中的城市
        placeSearch.search(e.poi.name, (status, result) => {
          if (status === 'complete' && result.info === 'OK') {
            const poi = result.poiList.pois[0];
            // console.log('poi---', poi, 'result---', result);
            const { cityname, adname, name, address } = poi;
            console.log('搜索位置---', cityname + adname + name);
            // 移除地图上所有自动生成的标记
            map.clearMap();
            map.setCenter([poi.location.lng, poi.location.lat]);
            setMarker(map, AMap, {
              longitude: poi.location.lng,
              latitude: poi.location.lat,
            });
          } else {
            console.error('搜索失败，请稍后再试');
          }
        });
      });

      setMarker(map, AMap, res);

      // 点击地图选择位置
      map.on('click', (e) => {
        const position = {
          longitude: e.lnglat.lng,
          latitude: e.lnglat.lat,
        };
        position.value = [e.lnglat.lng, e.lnglat.lat];
        setMarker(map, AMap, position);
        const lnglat = [e.lnglat.lng, e.lnglat.lat];
        // 根据坐标信息获取地址
        geocoder.value.getAddress(lnglat, (status, result) => {
          if (status === 'complete' && result.regeocode) {
            selectAddress.value = result.regeocode.formattedAddress;
            console.log(
              '手动选中地址信息：',
              selectAddress.value,
              '----',
              result,
            );
          } else {
            console.error('逆地理编码失败：', result);
          }
        });
      });
    })
    .catch((e) => {
      console.log(e);
    });
};

const setMarker = (map, AMap, position) => {
  if (marker) {
    map.remove(marker); // 移除指定 Marker
    marker = null; // 清除 Marker 引用
  }
  //创建一个 Marker 实例：
  marker = new AMap.Marker({
    position: new AMap.LngLat(position.longitude, position.latitude),
    title: '',
  });
  //将创建的点标记添加到已有的地图实例：
  map.add(marker);
};

//h5获取地理位置
const getLocationH5 = () => {
  return new Promise((resolve, reject) => {
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          const { latitude, longitude } = position.coords;
          // lag = longitude; //经度
          // lat = latitude; //纬度
          console.log('纬度:' + latitude + ',经度:' + longitude);
          resolve({
            latitude,
            longitude,
          });
        },
        (error) => {
          switch (error.code) {
            case error.PERMISSION_DENIED:
              reject('定位失败,用户拒绝请求地理定位');
              console.log('定位失败,用户拒绝请求地理定位');
              break;
            case error.POSITION_UNAVAILABLE:
              reject('定位失败,位置信息是不可用');

              console.log('定位失败,位置信息是不可用');
              break;
            case error.TIMEOUT:
              reject('定位失败,请求获取用户位置超时');

              console.log('定位失败,请求获取用户位置超时');
              break;
            case error.UNKNOWN_ERROR:
              reject('定位失败,定位系统失效');

              console.log('定位失败,定位系统失效');
              break;
          }
        },
        {
          enableHighAccuracy: true,
          timeout: 5000,
          maximumAge: 0,
          coordsType: 'gcj02', // 指定坐标系为GCJ-02
        },
      );
    } else {
      console.error('浏览器不支持地理定位。');
      reject('浏览器不支持地理定位。');
    }
  });
};

onMounted(() => {
  window._AMapSecurityConfig = {
    securityJsCode: '6f831b32c885d1b926b5327d723008fb', // 密钥
  };
  // getLocation();
  initMap();
});

onUnmounted(() => {
  map?.destroy();
});
</script>
<style scoped>
#container {
  width: 100%;
  height: 100vh;
}
.map-wrap{
  position: relative;
}
.search_input{
  position: absolute;
  top: 10px;
  right: 20px;
  z-index: 9999;

}
</style>
