---
title: Fragment懒加载
date: 2020-05-01 14:09:33
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
    - Android
---

## 懒加载

懒加载即在页面第一次可见时再进行数据请求以及加载的操作，如果不进行懒加载会导致多个fragment页面的生命周期被调用，每个页面都进行网络请求这样会产生很多无用的请求，因为实际显示的只是用户看到的那个页面，其他页面没有必要在这个时候去加载数据。

**懒加载两个使用场景**
场景一：使用FragmentManager实现底部导航栏页面切换
场景二：使用ViewPager实现标签栏页面切换

## 使用FragmentManager实现底部导航栏页面切换

**1.通过使用add()+hide()+show()方法来实现Fragment切换**，实现代码如下并打印生命周期日志:
```js
FragmentManager fragmentManager = getSupportFragmentManager();
        FragmentTransaction ft = fragmentManager.beginTransaction();
//        ft.addToBackStack(null);
        fragments[0] = new FirstFragment();
        ft.add(R.id.nav_host_fragment, fragments[0]);
        ft.commitAllowingStateLoss();

        navView.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem menuItem) {
                switch (menuItem.getItemId()) {
                    case R.id.navigation_user:
                        switchContent(getCurrentFragment(), fragments[0]);
                        return true;
                    case R.id.navigation_room:
                        if (fragments[1] == null) {
                            fragments[1] = new SecondFragment();
                            addContent(getCurrentFragment(), fragments[1]);
                        } else {
                            switchContent(getCurrentFragment(), fragments[1]);
                        }
                        return true;
                    case R.id.navigation_setting:
                        if (fragments[2] == null) {
                            fragments[2] = new ThirdFragment();
                            addContent(getCurrentFragment(), fragments[2]);
                        } else {
                            switchContent(getCurrentFragment(), fragments[2]);
                        }
                        return true;
                }
                return false;
            }
        });
    }

    public void switchContent(Fragment from, Fragment to) {
        FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
        transaction.hide(from).show(to).commitAllowingStateLoss();
    }

    public void addContent(Fragment from, Fragment to) {
        FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
        transaction.hide(from).add(R.id.nav_host_fragment, to).commitAllowingStateLoss();
    }
```
默认先打开Fragment_01,并依次切换Fragment_01->Fragment_02->Fragment_03->Fragment_01，最后关闭Activity
```js
打开默认第一个界面
Fragment_01 onAttach
Fragment_01 onCreate
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume

切换到Fragment_02界面
Fragment_02 onAttach
Fragment_02 onCreate
Fragment_02 onCreateView
Fragment_02 onStart
Fragment_02 onResume

切换到Fragment_03界面
Fragment_03 onAttach
Fragment_03 onCreate
Fragment_03 onCreateView
Fragment_03 onStart
Fragment_03 onResume

切换到Fragment_01界面
...
没有触发生命周期
...

关闭Activity
Fragment_01 onPause
Fragment_02 onPause
Fragment_03 onPause
Fragment_01 onStop
Fragment_02 onStop
Fragment_03 onStop
Fragment_01 onDestroyView
Fragment_01 onDestroy
Fragment_01 onDetach
Fragment_02 onDestroyView
Fragment_02 onDestroy
Fragment_02 onDetach
Fragment_03 onDestroyView
Fragment_03 onDestroy
Fragment_03 onDetach
```
根据上面的生命周期所示，通过add()+hide()+show()方式实现的Fragmnet切换，不可见的Fragment并不会销毁。

**2.使用replace()来实现Fragment界面的切换**,实现代码如下并打印生命周期日志:
```js
 FragmentManager fragmentManager = getSupportFragmentManager();
        FragmentTransaction ft = fragmentManager.beginTransaction();
        ft.setTransition(FragmentTransaction.TRANSIT_FRAGMENT_OPEN);
//        ft.addToBackStack(null);
        ft.add(R.id.nav_host_fragment, new FirstFragment());
        ft.commit();

        navView.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem menuItem) {
                switch (menuItem.getItemId()) {
                    case R.id.navigation_user:
                        switchContent( new FirstFragment());
                        return true;
                    case R.id.navigation_room:
                        switchContent(new SecondFragment());
                        return true;
                    case R.id.navigation_setting:
                        switchContent(new ThirdFragment());
                        return true;
                }
                return false;
            }
        });
    }

    public void switchContent(Fragment to) {
        if (currentFragment != to) {
            currentFragment = to;
            FragmentTransaction transaction = getSupportFragmentManager().beginTransaction();
            transaction.replace(R.id.nav_host_fragment, to);
//            transaction.addToBackStack(null);//会将Fragment添加到回退栈中
            transaction.commitAllowingStateLoss();
        }
    }
```
默认先打开Fragment_01,并依次切换Fragment_01->Fragment_02->Fragment_03->Fragment_01，最后关闭Activity
```js
打开默认第一个界面Fragment_01
Fragment_01 onAttach
Fragment_01 onCreate
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume

切换到Fragment_02界面
Fragment_02 onAttach
Fragment_02 onCreate
Fragment_01 onPause
Fragment_01 onStop
Fragment_01 onDestroyView
Fragment_01 onDestroy
Fragment_01 onDetach
Fragment_02 onCreateView
Fragment_02 onStart
Fragment_02 onResume

切换到Fragment_03界面
Fragment_03 onAttach
Fragment_03 onCreate
Fragment_02 onPause
Fragment_02 onStop
Fragment_02 onDestroyView
Fragment_02 onDestroy
Fragment_02 onDetach
Fragment_03 onCreateView
Fragment_03 onStart
Fragment_03 onResume

切换到Fragment_01界面
Fragment_01 onAttach
Fragment_01 onCreate
Fragment_03 onPause
Fragment_03 onStop
Fragment_03 onDestroyView
Fragment_03 onDestroy
Fragment_03 onDetach
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume

关闭Activity
Fragment_01 onPause
Fragment_01 onStop
Fragment_01 onDestroyView
Fragment_01 onDestroy
Fragment_01 onDetach
```
replace()方法实质是通过内部调用remove()和add()来实现Fragment的修改；通过这种方式切换界面，会销毁不可见的Fragment，重新创建当前的Fragmnent，虽然Fragment也是在可见时加载数据，当时会导致数据多次加载，浪费资源。

## 使用ViewPager实现标签栏页面切换

Viewpager的Adapter有两种:FragmentPagerAdapter和FragmentStatePagerAdapter。下面看一下这两种Adapter的区别：

**1.FragmentPagerAdapter**
```js
FragmentManager fragmentManager = getSupportFragmentManager();
        FragmentTransaction ft = fragmentManager.beginTransaction();
//        ft.addToBackStack(null);
        list.add(new FirstFragment());
        list.add(new SecondFragment());
        list.add(new ThirdFragment());

        FragmentPagerAdapter fragmentPagerAdapter = new FragmentPagerAdapter(getSupportFragmentManager()) {
            @NonNull
            @Override
            public Fragment getItem(int position) {
                return list.get(position);
            }

            @Override
            public int getCount() {
                return list.size();
            }
        };
        viewPager.setAdapter(fragmentPagerAdapter);
//        viewPager.setOffscreenPageLimit(2); //预加载剩下两页
        
        navView.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem menuItem) {
                switch (menuItem.getItemId()) {
                    case R.id.navigation_user:
                        viewPager.setCurrentItem(0);
                        return true;
                    case R.id.navigation_room:
                        viewPager.setCurrentItem(1);
                        return true;
                    case R.id.navigation_setting:
                        viewPager.setCurrentItem(2);
                        return true;
                }
                return false;
            }
        });
```
默认先打开Fragment_01,并依次切换Fragment_01->Fragment_02->Fragment_03->Fragment_01，最后关闭Activity
```js
打开默认第一个界面Fragment_01
Fragment_01 onAttach
Fragment_01 onCreate
Fragment_02 onAttach
Fragment_02 onCreate
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume
Fragment_02 onCreateView
Fragment_02 onStart
Fragment_02 onResume

切换到Fragment_02界面
Fragment_03 onAttach
Fragment_03 onCreate
Fragment_03 onCreateView
Fragment_03 onStart
Fragment_03 onResume

切换到Fragment_03界面
Fragment_01 onPause
Fragment_01 onStop
Fragment_01 onDestroyView

切换到Fragment_01界面
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume
Fragment_03 onPause
Fragment_03 onStop
Fragment_03 onDestroyView

关闭Activity
Fragment_02 onPause
Fragment_01 onPause
Fragment_02 onStop
Fragment_01 onStop
Fragment_02 onDestroyView
Fragment_02 onDestroy
Fragment_02 onDetach
Fragment_01 onDestroyView
Fragment_01 onDestroy
Fragment_01 onDetach
Fragment_03 onDestroy
Fragment_03 onDetach
```
通过FragmentPagerAdapter方式，销毁Fragment时只是销毁了Fragment的视图，并没有执行onDestory()方法。

**2.FragmentStatePagerAdapter**
```js
 FragmentManager fragmentManager = getSupportFragmentManager();
        FragmentTransaction ft = fragmentManager.beginTransaction();
//        ft.addToBackStack(null);
        list.add(new FirstFragment());
        list.add(new SecondFragment());
        list.add(new ThirdFragment());

        FragmentStatePagerAdapter fragmentPagerAdapter = new FragmentStatePagerAdapter(getSupportFragmentManager()) {
            @NonNull
            @Override
            public Fragment getItem(int position) {
                return list.get(position);
            }

            @Override
            public int getCount() {
                return list.size();
            }
        };
        viewPager.setAdapter(fragmentPagerAdapter);
//        viewPager.setOffscreenPageLimit(2); //预加载剩下两页

        navView.setOnNavigationItemSelectedListener(new BottomNavigationView.OnNavigationItemSelectedListener() {
            @Override
            public boolean onNavigationItemSelected(@NonNull MenuItem menuItem) {
                switch (menuItem.getItemId()) {
                    case R.id.navigation_user:
                        viewPager.setCurrentItem(0);
                        return true;
                    case R.id.navigation_room:
                        viewPager.setCurrentItem(1);
                        return true;
                    case R.id.navigation_setting:
                        viewPager.setCurrentItem(2);
                        return true;
                }
                return false;
            }
        });
```
默认先打开Fragment_01,并依次切换Fragment_01->Fragment_02->Fragment_03->Fragment_01，最后关闭Activity
```js
打开默认第一个界面Fragment_01
Fragment_01 onAttach
Fragment_01 onCreate
Fragment_02 onAttach
Fragment_02 onCreate
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume
Fragment_02 onCreateView
Fragment_02 onStart
Fragment_02 onResume

切换到Fragment_02界面
Fragment_03 onAttach
Fragment_03 onCreate
Fragment_03 onCreateView
Fragment_03 onStart
Fragment_03 onResume

切换到Fragment_03界面
Fragment_01 onPause
Fragment_01 onStop
Fragment_01 onDestroyView
Fragment_01 onDestroy
Fragment_01 onDetach

切换到Fragment_01界面
Fragment_01 onAttach
Fragment_01 onCreate
Fragment_01 onCreateView
Fragment_01 onStart
Fragment_01 onResume
Fragment_03 onPause
Fragment_03 onStop
Fragment_03 onDestroyView
Fragment_03 onDestroy
Fragment_03 onDetach

关闭Activity
Fragment_02 onPause
Fragment_01 onPause
Fragment_02 onStop
Fragment_01 onStop
Fragment_02 onDestroyView
Fragment_02 onDestroy
Fragment_02 onDetach
Fragment_01 onDestroyView
Fragment_01 onDestroy
Fragment_01 onDetach

```
通过FragmentStatePagerAdapter方式，销毁Fragment时不仅销毁了Fragment的视图，还销毁了Fragment对象，执行了onDestory()和onDetach()

>上述几种方式，要根据实际情况选择使用

[项目地址](https://gitee.com/jags/FragmentLoad.git)