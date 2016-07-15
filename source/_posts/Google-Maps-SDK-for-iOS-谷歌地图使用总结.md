---
title: Google Maps SDK for iOS 谷歌地图使用总结
date: 2016-05-17 21:58:42
tags:
---
由于公司项目的需求，我们的项目需要集成谷歌和百度两套地图，国内用百度国外用谷歌，在这里记录一下谷歌地图的常用方法。
<!--more-->
# 集成步骤
SDK下载集成步骤,[请参照这篇文章](http://www.jianshu.com/p/dc7d267d63d0)
# 常用类介绍
- GMSMapView 最主要的地图类  
- GMSCameraPosition 地图摄像头，可以理解为当前地图的可视范围，可以获取到摄像头中心点坐标、镜头缩放比例、方向、视角等参数
- GMSMarker 地图大头针
- GMSGeocoder 反向地理编码类
- GMSAddress 反向地理编码返回的类,包含坐标及地理位置描述等信息
- CLLocationManager 就是CoreLocation框架下的地理位置管理类
- GMSAutocompleteFetcher 搜索自动补全抓取器，通过该类的代理方法实现搜索自动补全
# 常用的方法
## GMSMapView
- [mapView:willMove:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#ae1f53e0d056bf4d246356134ec594612) 镜头即将移动时调用
- [mapView:didChangeCameraPosition:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#aabd01d59d7680799a0c24d3c8b5e4622) 镜头移动完成后调用
- [mapView:didTapAtCoordinate:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#a9f226252840c79a996df402da9eec235) 点击地图时调用
- [mapView:didLongPressAtCoordinate:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#a0499f484d9ea51041babd3ac8dcc7b27) 长按地图时调用
- [mapView:didTapMarker:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#ae291e1bff99f9d06428ced09daf5802d)  点击大头针时调用
- [mapView:didTapInfoWindowOfMarker:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#ae1614978aef680d671a6a95da8e13bf7) 点击大头针的弹出视窗时调用
- [mapView:didLongPressInfoWindowOfMarker:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#a4ff47da211c1cc6ca0aa53d52717f0db) 长按大头针视窗时调用
- [mapView:markerInfoWindow:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#ab83c3dab588f06e6227794740782bf55) 自定义大头针弹出视窗，返回UIView
- [mapView:didDragMarker:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#ae13d024d818081757bb058ecc532459a) 拖拽大头针时调用
- [mapView:didEndDraggingMarker:](https://developers.google.com/maps/documentation/ios-sdk/reference/protocol_g_m_s_map_view_delegate-p.html#adab19737cc05cfda53a03c0247ab9e1e) 大头针拖拽完成时调用
## GMSAutocompleteFetcher
- sourceTextHasChanged: 传入要搜索的文字
- didAutocompleteWithPredictions: 根据输入文字，得到自动补全的结果（数组类型）
- didFailAutocompleteWithError: 错误捕捉
*需要注意的是，要使用自动补全API，需要在谷歌后台创建支持Google Places API的Key，而且普通用户每天只能免费试用1000次*
## Reverse GMSGeocoder 反向地理编码
- reverseGeocodeCoordinate:completionHandler: 反向地理编码，把经纬度转换为地理位置
## 地理编码
需要特别说明的是，谷歌地图的地理编码还是有点反人类的，需要自己构造一个网络请求，然后自己做相应处理。谷歌的地理编码用到的是[
                                                              Google Maps Geocoding API](https://developers.google.com/maps/documentation/geocoding/intro#geocoding) 首先需要去谷歌的开发者后台激活Google Maps Geocoding API，Key需要作为一个参数拼接到网络请求后面。
例如:https://maps.googleapis.com/maps/api/geocode/json?address= Staples Center&key=API_KEY
需要注意的是，普通用户API每天只能进行2500次请求，想要增加配合需要付费。
# Demo
包含以上功能的小demo,[点此查看](https://github.com/WhisperKarl/GoogleMaps-Demo)
运行前请先pod install一下, 记得翻墙..~