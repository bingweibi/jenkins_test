// Copyright 2015 The Chromium Authors. All rights reserved.
// Use of this source code is governed by a BSD-style license that can be
// found in the LICENSE file.

import 'dart:async';

import 'package:extended_tabs/src/page_view.dart';
import 'package:flutter/gestures.dart';
import 'package:flutter/material.dart';
import 'package:flutter/rendering.dart';
import 'package:flutter/widgets.dart';

/// A page view that displays the widget which corresponds to the currently
/// selected tab.
///
/// This widget is typically used in conjunction with a [TabBar].
///
/// If a [TabController] is not provided, then there must be a [DefaultTabController]
/// ancestor.
///
/// The tab controller's [TabController.length] must equal the length of the
/// [children] list and the length of the [TabBar.tabs] list.
///
/// To see a sample implementation, visit the [TabController] documentation.
class ExtendedTabBarView extends StatefulWidget {
  /// Creates a page view with one child per tab.
  ///
  /// The length of [children] must be the same as the [controller]'s length.
  const ExtendedTabBarView(
      {Key key,
      @required this.children,
      this.controller,
      this.physics,
      this.dragStartBehavior = DragStartBehavior.start,
      this.cacheExtent = 0,
      this.linkWithAncestor = true})
      : assert(children != null),
        assert(dragStartBehavior != null),
        super(key: key);

  /// cache page count
  /// default is 0.
  /// if cacheExtent is 1, it has two pages in cache
  /// null is infinity, it will cache all pages
  final int cacheExtent;

  ///if linkedParentTabBarView is true and current tabbarview over scroll,
  ///it will check whether ancestor tabbarView can be scroll
  ///then scroll ancestor tabbarView
  final bool linkWithAncestor;

  /// This widget's selection and animation state.
  ///
  /// If [TabController] is not provided, then the value of [DefaultTabController.of]
  /// will be used.
  final TabController controller;

  /// One widget per tab.
  ///
  /// Its length must match the length of the [TabBar.tabs]
  /// list, as well as the [controller]'s [TabController.length].
  final List<Widget> children;

  /// How the page view should respond to user input.
  ///
  /// For example, determines how the page view continues to animate after the
  /// user stops dragging the page view.
  ///
  /// The physics are modified to snap to page boundaries using
  /// [PageScrollPhysics] prior to being used.
  ///
  /// Defaults to matching platform conventions.
  final ScrollPhysics physics;

  /// {@macro flutter.widgets.scrollable.dragStartBehavior}
  final DragStartBehavior dragStartBehavior;

  @override
  _ExtendedTabBarViewState createState() => _ExtendedTabBarViewState();
}

final PageScrollPhysics _kTabBarViewPhysics =
    const PageScrollPhysics().applyTo(const ClampingScrollPhysics());

class _ExtendedTabBarViewState extends State<ExtendedTabBarView> {
  TabController _controller;
  PageController _pageController;
  _ExtendedTabBarViewState _ancestor;
  List<Widget> _children;
  List<Widget> _childrenWithKey;
  int _currentIndex;
  int _warpUnderwayCount = 0;

  // If the TabBarView is rebuilt with a new tab controller, the caller should
  // dispose the old one. In that case the old controller's animation will be
  // null and should not be accessed.
  bool get _controllerIsValid => _controller?.animation != null;

  void _updateTabController() {
    final TabController newController =
        widget.controller ?? DefaultTabController.of(context);
    assert(() {
      if (newController == null) {
        throw FlutterError('No TabController for ${widget.runtimeType}.\n'
            'When creating a ${widget.runtimeType}, you must either provide an explicit '
            'TabController using the "controller" property, or you must ensure that there '
            'is a DefaultTabController above the ${widget.runtimeType}.\n'
            'In this case, there was neither an explicit controller nor a default controller.');
      }
      return true;
    }());

    if (newController == _controller) return;

    if (_controllerIsValid)
      _controller.animation.removeListener(_handleTabControllerAnimationTick);
    _controller = newController;
    if (_controller != null)
      _controller.animation.addListener(_handleTabControllerAnimationTick);
  }

  @override
  void initState() {
    super.initState();
    _updateChildren();
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    _updateTabController();
    _currentIndex = _controller?.index;
    _pageController = PageController(initialPage: _currentIndex ?? 0);
  }

  @override
  void didUpdateWidget(ExtendedTabBarView oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.controller != oldWidget.controller) _updateTabController();
    if (widget.children != oldWidget.children && _warpUnderwayCount == 0)
      _updateChildren();
  }

  @override
  void dispose() {
    if (_controllerIsValid)
      _controller.animation.removeListener(_handleTabControllerAnimationTick);
    _controller = null;
    // We don't own the _controller Animation, so it's not disposed here.
    super.dispose();
  }

  void _updateChildren() {
    _children = widget.children;
    _childrenWithKey = KeyedSubtree.ensureUniqueKeysForList(widget.children);
  }

  void _handleTabControllerAnimationTick() {
    if (_warpUnderwayCount > 0 || !_controller.indexIsChanging)
      return; // This widget is driving the controller's animation.

    if (_controller.index != _currentIndex) {
      _currentIndex = _controller.index;
      _warpToCurrentIndex();
    }
  }

  Future<void> _warpToCurrentIndex() async {
    if (!mounted) return Future<void>.value();

    if (_pageController.page == _currentIndex.toDouble())
      return Future<void>.value();

    final int previousIndex = _controller.previousIndex;
    if ((_currentIndex - previousIndex).abs() == 1)
      return _pageController.animateToPage(_currentIndex,
          duration: kTabScrollDuration, curve: Curves.ease);

    assert((_currentIndex - previousIndex).abs() > 1);
    final int initialPage =
        _currentIndex > previousIndex ? _currentIndex - 1 : _currentIndex + 1;
    final List<Widget> originalChildren = _childrenWithKey;
    setState(() {
      _warpUnderwayCount += 1;

      _childrenWithKey = List<Widget>.from(_childrenWithKey, growable: false);
      final Widget temp = _childrenWithKey[initialPage];
      _childrenWithKey[initialPage] = _childrenWithKey[previousIndex];
      _childrenWithKey[previousIndex] = temp;
    });
    _pageController.jumpToPage(initialPage);

    await _pageController.animateToPage(_currentIndex,
        duration: kTabScrollDuration, curve: Curves.ease);
    if (!mounted) return Future<void>.value();
    setState(() {
      _warpUnderwayCount -= 1;
      if (widget.children != _children) {
        _updateChildren();
      } else {
        _childrenWithKey = originalChildren;
      }
    });
  }

  bool flag = true;
  // Called when the PageView scrolls
  bool _handleScrollNotification(ScrollNotification notification) {
    if (_warpUnderwayCount > 0) return false;

    if (notification.depth != 0) return false;

    _warpUnderwayCount += 1;
    if (notification is ScrollUpdateNotification &&
        !_controller.indexIsChanging) {
      if ((_pageController.page - _controller.index).abs() > 1.0) {
        _controller.index = _pageController.page.floor();
        _currentIndex = _controller.index;
      }
      _controller.offset =
          (_pageController.page - _controller.index).clamp(-1.0, 1.0);
    } else if (notification is ScrollEndNotification) {
      _controller.index = _pageController.page.round();
      _currentIndex = _controller.index;
    } else if (notification is OverscrollNotification && _ancestor != null) {
      if (_canlinkeWithAncestorScroll(notification.overscroll < 0)) {
          print("===初始> ${_ancestor._pageController.offset}");
          _ancestor._pageController.jumpTo( helpOffset(_ancestor._pageController.offset,notification.overscroll));
//            .moveTo(_ancestor._pageController.offset + notification.overscroll);
      }
    }
    _warpUnderwayCount -= 1;

    return false;
  }

  double helpOffset(double a,double b){
    print("===%%%%%%  ${flag}");
    if(a==360 && flag){
      if(b>0){
        return (a+360).clamp(360.0,720.0);
      }else{
        return (a-360).clamp(0.0,360.0);
      }
    }else if(a==0&&b>0){
      flag = false;
      return (a+360).clamp(0.0,360.0);
    }
    return 0;
  }

  bool _handleGlowNotification(OverscrollIndicatorNotification notification) {
    if (notification.depth == 0 &&
        _canlinkeWithAncestorScroll(notification.leading)) {
      notification.disallowGlow();
      return true;
    }
    return false;
  }

  bool _canlinkeWithAncestorScroll(bool onLeftEdge) {
    //return false;
    if (_ancestor == null) return false;
    return (onLeftEdge &&
            _ancestor._pageController.offset !=
                _ancestor._pageController.position.minScrollExtent) ||
        ((!onLeftEdge &&
            _ancestor._pageController.offset !=
                _ancestor._pageController.position.maxScrollExtent));
  }

  @override
  Widget build(BuildContext context) {
    assert(() {
      if (_controller.length != widget.children.length) {
        throw FlutterError(
            'Controller\'s length property (${_controller.length}) does not match the \n'
            'number of tabs (${widget.children.length}) present in TabBar\'s tabs property.');
      }
      return true;
    }());
    if (widget.linkWithAncestor) {
      _ancestor =
          context.ancestorStateOfType(TypeMatcher<_ExtendedTabBarViewState>());
    }
    return NotificationListener<ScrollNotification>(
      onNotification: _handleScrollNotification,
      child: NotificationListener<OverscrollIndicatorNotification>(
        onNotification: _handleGlowNotification,
        child: ExtendedPageView(
          dragStartBehavior: widget.dragStartBehavior,
          controller: _pageController,
          cacheExtent: widget.cacheExtent,
          physics: widget.physics == null
              ? _kTabBarViewPhysics
              : _kTabBarViewPhysics.applyTo(widget.physics),
          children: _childrenWithKey,
        ),
      ),
    );
  }
}

