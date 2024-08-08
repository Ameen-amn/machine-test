# machine-test
import 'dart:async';

import 'package:aqary_week_assessment_task/core/constants_color.dart';
import 'package:aqary_week_assessment_task/core/data/api.dart';
import 'package:aqary_week_assessment_task/core/enum.dart';
import 'package:aqary_week_assessment_task/core/widget/custom_error_screen.dart';
import 'package:aqary_week_assessment_task/core/widget/custom_loader.dart';
import 'package:aqary_week_assessment_task/google_map/presentation/bloc/gm_bloc.dart';
import 'package:aqary_week_assessment_task/google_map/presentation/widget/alert_widget.dart';
import 'package:flutter/material.dart';
import 'package:flutter_bloc/flutter_bloc.dart';
import 'package:flutter_map/flutter_map.dart';
import 'package:geolocator/geolocator.dart';
import 'package:latlong2/latlong.dart';

class MapScreen extends StatefulWidget {
  final TabController tabController;
  const MapScreen({super.key, required this.tabController});

  @override
  State<MapScreen> createState() => _MapScreenState();
}

class _MapScreenState extends State<MapScreen> {
  MapController mapController = MapController();
  List<List<LatLng>> _animatedPointsList = [];
  final int _currentIndex = 0;
  Timer? _timer;
  double currentZoom = 8;
  final int _currentStep = 0;
  double _fraction = 0.0;
  List<List<LatLng>>? _launchPoints;
  LatLng center = const LatLng(51.509364, -0.128928);
  // double currentZoom = 13.0;

  @override
  void initState() {
    context.read<GmBloc>().add(const GmEvent.fetchLaunchDataToMap());

    // _launchPoints = launchMockList;
    // print('launch$launchMockList');
    // _animatePolyline2(launchMockList);
    // .map((item) => _animatePolyline(item));
    super.initState();
  }

  @override
  void dispose() {
    _timer?.cancel();
    super.dispose();
  }

  void _zoomIn() {
    currentZoom = currentZoom + 1;
    mapController.move(center, currentZoom);
  }

  void _zoom() {
    currentZoom = currentZoom - 1;
    mapController.move(center, currentZoom);
  }

  // void _animatePolyline(List<LaunchMock> launch) {
  //   Timer.periodic(const Duration(seconds: 1), (Timer timer) {
  //     for(var item in launch){
  //       _animatedPoints.
  //     }
  //     if (_animatedPoints.length < launch.length) {
  //       setState(() {
  //         _animatedPoints.map((polyLine) {
  //           polyLine.add(points[polyLine.length]);
  //         });
  //       });
  //     } else {
  //       timer.cancel();
  //     }
  //   });
  // }

  LatLng _interpolate(LatLng start, LatLng end, double fraction) {
    return LatLng(
      start.latitude + (end.latitude - start.latitude) * fraction,
      start.longitude + (end.longitude - start.longitude) * fraction,
    );
  }

  void _initializeAnimatedPoints() {
    if (_launchPoints != null) {
      _animatedPointsList = List.generate(_launchPoints!.length, (index) => []);
    }
  }

  void _animatePolyline2() {
    print('helo');
    const int steps = 100;
    const Duration stepDuration = Duration(seconds: 1);
    _timer = Timer.periodic(stepDuration, (Timer timer) {
      print('laun $_launchPoints');

      _fraction += 1.0 / steps;
      if (_fraction > 1.0) {
        _fraction = 1.0;
      }
      setState(() {
        for (int i = 0; i < _launchPoints!.length; i++) {
          final start = _launchPoints![i][0];
          final end = _launchPoints![i][1];
          _animatedPointsList[i].add(_interpolate(start, end, _fraction));
        }
      });

      if (_fraction >= 1.0) {
        _fraction = 0.0;
        _initializeAnimatedPoints();
      }
    });
  }


  @override
  Widget build(BuildContext context) {
    return BlocConsumer<GmBloc, GmState>(
      listener: (context, state) {
        final launchMockList = context
            .read<GmBloc>()
            .state
            .launchMapMarks
            .map((item) => [item.launch, item.landing])
            .toList();
        _launchPoints = launchMockList;
        _initializeAnimatedPoints();
        _animatePolyline2();
        if (!state.serviceEnabled && state.getCurrentLocation) {
          ScaffoldMessenger.of(context).showSnackBar(const SnackBar(
              content: Text(
                  'Location services are disabled. Please enable the services')));
        } else if (state.locationPermission == LocationPermission.denied) {
          ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('Location permissions are denied')));
        } else if (state.locationPermission ==
            LocationPermission.deniedForever) {
          ScaffoldMessenger.of(context).showSnackBar(const SnackBar(
              content: Text(
                  'Location permissions are permanently denied, we cannot request permissions.')));
        }
        if (state.showWarning) {
          showDialog(
              context: context,
              builder: (_) => const CustomAlertDialog(
                  textMessage: 'You are near a Launch site'));
        }
      },
      builder: (context, state) {
        // final List<List<LatLng>> launchMockList = state.launchMapMarks
        //     .map((item) => [item.launch, item.landing])
        //     .toList();
        // _launchPoints = launchMockList;

        // print('launch$launchMockList');
        if (state.isLoading) {
          return const CustomLoader();
        }
        if (state.errorOccured && state.isLoaded) {
          return const CustomErrorScreen(error: 'Error Occured');
        }
        if (state.isLoaded && !state.isLoading && !state.errorOccured) {
          return Stack(
            children: [
              FlutterMap(
                options: MapOptions(
                  onMapEvent: (e) {},
                  initialCenter: const LatLng(51.509364, -0.128928),
                  initialZoom: 8,
                  onMapReady: () async {},
                ),
                mapController: mapController,
                children: [
                  TileLayer(
                      tileBuilder: (context, tileWidget, tile) {
                        return ColorFiltered(
                            colorFilter: const ColorFilter.matrix(<double>[
                              0.2126,
                              0.7152,
                              0.0722,
                              0,
                              0,
                              0.2126,
                              0.7152,
                              0.0722,
                              0,
                              0,
                              0.2126,
                              0.7152,
                              0.0722,
                              0,
                              0,
                              0,
                              0,
                              0,
                              1,
                              0
                            ]),
                            child: tileWidget);
                      },
                      tileDisplay: const TileDisplay.fadeIn(),
                      urlTemplate: mapApi,
                      userAgentPackageName: 'com.example.app'),
                  PolylineLayer(
                    polylines: _animatedPointsList.map((points) {
                      return Polyline(
                          points: points, strokeWidth: 4.0, color: Colors.blue);
                    }).toList(),
                  ),
                  MarkerLayer(markers: [
                    ...state.launchMapMarks
                        .map((item) => Marker(
                            point: item.launch,
                            child: Icon(Icons.location_on_sharp,
