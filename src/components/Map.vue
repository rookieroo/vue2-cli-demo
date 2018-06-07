<template>
  <div>
    <v-distpicker @selected="searchKeyword"></v-distpicker>
    <div style="width:750px;height:350px;margin-left: 50px" id="container"></div>
  </div>
</template>

<script>
  import VDistpicker from 'v-distpicker'

  export default {
    name: 'ElectricFence',
    components: {VDistpicker},
    data() {
      return {
        map: null,
        overlaysArray: [], // 覆盖物容器,用于清除覆盖物
        radius: 1000,
        points: '',
        add: this.OssUrl + '/sup/website/add',
        center: null,
        labelPosition: 'right',
        form: {
          orgName: '',
          orgNo: '',
          dis: null
        },
        rules: {
          orgName: [{required: true, message: '机构名称为必填项', trigger: 'blur'}],
          orgNo: [{required: true, message: '机构号为必选项', trigger: 'blur'}]
        },
      }
    },
    mounted() {
      this.createMap()
    },
    watch: {},
    methods: {
      createMap() {
        let _this = this
        _this.map = new qq.maps.Map('container', {
          center: new qq.maps.LatLng(39.916527, 116.397128),
          zoom: 14
        });
        let drawingManager = new qq.maps.drawing.DrawingManager({
          drawingMode: qq.maps.drawing.OverlayType.POLYGON,
          drawingControl: true,
          drawingControlOptions: {
            position: qq.maps.ControlPosition.TOP_CENTER,
            drawingModes: [
              qq.maps.drawing.OverlayType.MARKER,
              qq.maps.drawing.OverlayType.CIRCLE,
              qq.maps.drawing.OverlayType.POLYGON,
              qq.maps.drawing.OverlayType.RECTANGLE
            ]
          },
          markerOptions: {
            visible: false
          },
          circleOptions: {
            fillColor: new qq.maps.Color(255, 208, 70, 0.3),
            strokeColor: new qq.maps.Color(88, 88, 88, 1),
            strokeWeight: 3,
            clickable: false
          }
        });
        drawingManager.setMap(_this.map);
        qq.maps.event.addListener(drawingManager, 'overlaycomplete', function (event) {
          if (event.type === "marker") {
            _this.clearOverlays(_this.overlaysArray);
            let latLng = event.overlay.getPosition();
            let lat = latLng.getLat().toFixed(5);
            let lng = latLng.getLng().toFixed(5);
            let center = new qq.maps.LatLng(lat, lng);
            let geocoder = new qq.maps.Geocoder();
            geocoder.getAddress(latLng);
            //设置服务请求成功的回调函数
            geocoder.setComplete(function (result) {
              _this.doMarker(center, result.detail.address + latLng);
              _this.doCircle(center);
              _this.getPeopleDataByCircle(_this.radius, lat + "," + lng);

            });
            //若服务请求失败，则运行以下函数
            geocoder.setError(function () {
              alert("出错了，请输入正确的经纬度！！！");
            });
          } else if (event.type === "circle") {
            _this.clearOverlays(_this.overlaysArray);
            _this.overlaysArray.push(event.overlay);
            let latLng = event.overlay.getCenter();
            let newRadius = event.overlay.getRadius();
            let lat = latLng.getLat().toFixed(5);
            let lng = latLng.getLng().toFixed(5);
            let center = new qq.maps.LatLng(lat, lng);
            let geocoder = new qq.maps.Geocoder();
            geocoder.getAddress(latLng);
            //设置服务请求成功的回调函数
            geocoder.setComplete(function (result) {
              _this.doMarker(center, result.detail.address + latLng);
              _this.getPeopleDataByCircle(newRadius, lat + "," + lng);

            });
            //若服务请求失败，则运行以下函数
            geocoder.setError(function () {
              alert("出错了，请输入正确的经纬度！！！");
            });

          } else if (event.type === "polygon" || event.type === "rectangle") {
            _this.clearOverlays(_this.overlaysArray);
            _this.overlaysArray.push(event.overlay);
            _this.points = ''
            event.overlay.getPath().forEach(function (e) {
              let lng = e.getLng();
              let lat = e.getLat();
              _this.points += lng + " " + lat + ",";
            });
            _this.points = _this.points.substring(0, _this.points.length - 1);
            // _this.getPeopleDataByPolygon(_this.points);
          }
        })
      },
      searchKeyword(data) {
        let _this = this
        let keyword = data.area.value + data.city.value + data.province.value
        let geocoder = new qq.maps.Geocoder({
          complete: function (result) {
            _this.map.setCenter(result.detail.location);
            var marker = new qq.maps.Marker({
              map: _this.map,
              position: result.detail.location
            });
          }
        });
        geocoder.getLocation(keyword);
      },
      onSubmit() {
        this.$refs['form'].validate((valid) => {
          if (valid) {
            let points = []
            points = this.points.split(',')
            for (let item of points) {
              let params = {
                'orgNo': this.form.orgNo,
                'orgName': this.form.orgName,
                'longitude': item.split(' ')[0],
                'latitude': item.split(' ')[1]
              }
              this.$axios.post(this.add, params).then((res) => {
                if (res.data.retCode == '1') {
                  this.resetForm()
                  this.$message({
                    message: '创建成功',
                    type: 'success'
                  })
                } else {
                  this.$message.error(res.data.retMessage);
                }
              })
            }
          }
        })
      },
      resetForm() {
        this.form.orgNo = ''
        this.form.orgName = ''
      },
      clearOverlays(overlaysArray) {
        if (overlaysArray) {
          for (let i in overlaysArray) {
            overlaysArray[i].setMap(null);
          }
        }
      },
      doMarker(center, title) {
        let marker = new qq.maps.Marker({
          position: center,
          map: this.map,
          title: title
        });
        marker.setVisible(true);
        marker.setAnimation(qq.maps.MarkerAnimation.DOWN);
        this.overlaysArray.push(marker);
        marker.setMap(this.map);
      },
      doCircle(center) {
        let circle = new qq.maps.Circle({
          map: this.map,
          center: center,
          radius: this.radius,
          strokeWeight: 5
        });
        this.overlaysArray.push(circle);
        circle.setMap(this.map);
      },
      getPeopleDataByCircle(radius, center) {
        alert("圆形中心为:" + center + "半径为:" + radius);
      },
      getPeopleDataByPolygon(points) {
        alert("多边形路径为:" + points);
      },
      // 默认浏览器客户端IP定位
      IsPtInPoly(ALon, ALat, APoints) {
        let iSum = 0,
          iCount;
        let dLon1, dLon2, dLat1, dLat2, dLon;
        if (APoints.length < 3) {
          return false
        }
        iCount = APoints.length;
        for (var i = 0; i < iCount; i++) {
          if (i == iCount - 1) {
            dLon1 = APoints[i].lng
            dLat1 = APoints[i].lat
            dLon2 = APoints[0].lng
            dLat2 = APoints[0].lat
          } else {
            dLon1 = APoints[i].lng
            dLat1 = APoints[i].lat
            dLon2 = APoints[i + 1].lng
            dLat2 = APoints[i + 1].lat
          }
          //以下语句判断A点是否在边的两端点的水平平行线之间，在则可能有交点，开始判断交点是否在左射线上
          if (((ALat >= dLat1) && (ALat < dLat2)) || ((ALat >= dLat2) && (ALat < dLat1))) {
            if (Math.abs(dLat1 - dLat2) > 0) {
              //得到 A点向左射线与边的交点的x坐标：
              dLon = dLon1 - ((dLon1 - dLon2) * (dLat1 - ALat)) / (dLat1 - dLat2);
              if (dLon < ALon)
                iSum++;
            }
          }
        }
        if (iSum % 2 != 0) {
          return true
        } else {
          return false
        }
      }
    }
  }
</script>

<style scoped>
  #container {
    width: 600px;
    height: 350px;
  }
</style>
