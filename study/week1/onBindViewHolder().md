## onBindViewHolder() 
끝까지 한번에 호출되는 문제
=> NestedScrollView 안에 Recyclerview가 있으면 안됨. 무조건 이 이슈가 발생함.

## NestedScrollView를 뺐는데도 이 이슈 발생
FragmentHeightWrappingViewPager 라는 CustomViewPager를 사용하고 있었음. 이것은 viewpager 안의 fragment 크기가 달라 좌우로 스크롤했을 때 짧은 프래그먼트가 빈공간인 것처럼 보임. 이 이슈를 해결하기 위해 넣었던 viewpager.

FragmentHeightWrappingViewPager
```java
 @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {

        if (mCurrentView == null) {
            super.onMeasure(widthMeasureSpec, heightMeasureSpec);
            return;
        }
        int height = 0;
        mCurrentView.measure(widthMeasureSpec, MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED));
        int h = mCurrentView.getMeasuredHeight();
        if (h > height) height = h;
        heightMeasureSpec = MeasureSpec.makeMeasureSpec(height, MeasureSpec.EXACTLY);
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
    }

    public void measureCurrentView(View currentView) {
        mCurrentView = currentView;
        requestLayout();
    }

    public int measureFragment(View view) {
        if (view == null)
            return 0;

        view.measure(0, 0);
        return view.getMeasuredHeight();
    }

```

FragmentStatePagerAdapter 
```java

    @Override
    public void setPrimaryItem(ViewGroup container, int position, Object object) {
        super.setPrimaryItem(container, position, object);
       if (position != mCurrentPosition) {
           Fragment fragment = (Fragment) object;
           FragmentHeightWrappingViewPager pager = (FragmentHeightWrappingViewPager) container;
           if (fragment != null && fragment.getView() != null) {
               mCurrentPosition = position;
               pager.measureCurrentView(fragment.getView());
           }
       }
    }
```

# 해결
- CoordinatorLayout + AppbarLayout + ViewPager 로 해결
- scroll

## Recyclerview 데이터 꼬이는 오류
가끔 position 0에 표시되어야 할 것이 1에 표시되어야하는 버그가 있었음.

### 이유 
Retrofit 비동기 통신을 할 때 position값을 final로 했음.

### 해결책
final로 하지말고 
holder.getAdapterPosition()로 position값을 가져와라.

### 주의점
- RecyclerView.NO_POSITION인지 꼭 확인하도록 해라.
- if(holder.getAdapterPosition()!=RecyclerView.NO_POSITION)


CoordinatorLayout works by searching through any child view that has a CoordinatorLayout Behavior defined either statically as XML with a app:layout_behavior tag or programmatically with the View class annotated with the @DefaultBehavior decorator. When a scroll event happens, CoordinatorLayout attempts to trigger other child views that are declared as dependencies.


# 참고문헌
https://medium.com/@bansooknam/android-recyclerview-%EC%9A%94%EC%95%BD-aaea4a9c95e7

# CoordinatorLayout
- CoordinatorLayout는 NestedScrollView가 스크롤시 layout_behavior에 정의된 레이아웃으로 스크롤 정보를 전달 하는 역할
-  AppBarLayout의 ScrollingViewBehavior가 정보를 받아서 AppBarLayout 자신을 변형
-  기존의 ScrollView나 ListView는 NestedScrollingChild가 구현되어 있지 않아 Behavior를 통해 스크롤 정보전달이 되지 않음

![이미지](http://androcode.es/wp-content/uploads/2015/10/simple_coordinator.gif)

## 기대한것처럼  동작하지 않은 fling 


 ## 참고문헌
 http://www.kmshack.kr/2017/01/coordinatorlayout%EA%B3%BC-behavior%EC%9D%98-%EA%B4%80%EA%B3%84/