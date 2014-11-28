---
layout: post
title: cocos 2dx 3.x实现的PageView
category: cocos2dx
date: 2014-11-27
---
###转载请注明出处：   
* [http://helkyle.tk/cocos-2dx-3xPageView/][1]
* [http://blog.csdn.net/joueu/article/details/41532763][2]  

> 之前使用PageView 觉得用起来好像不太爽，没办法达到我想要实现的功能，又不想修改源码。最近闲得蛋疼就花了半天时间捣鼓一个出来，就命名叫XKPageView吧，XK嘛...大家好像都这么命名~  
> 废话不多说，先看显示效果  
!["test"](/public/images/cocos-2dx_3x_PageView/test.gif)

> 因为是在模拟器上录像的，所以看上去会有点卡...真机测试就不会了。
（编号为5那个怎么会那么大嗟？) 看代码...

---
 
我直接上代码啦，代码里面都有注释了，一些太简单的东西就不多啰嗦。如果有问题的话可以留言问我哈~
,没有时间测试那么多...所以可能会有不少bug

PageView.h

	//  XKPageView.h
	//  XKPageView
	//
	//  Created by Joueu on 14-11-26.
	//
	//
	
	#ifndef __XKPageView__XKPageView__
	#define __XKPageView__XKPageView__
	
	#include "cocos2d.h"
	#include "cocos-ext.h"
	
	USING_NS_CC;
	USING_NS_CC_EXT;
	
	class XKPageView;
	class XKPageViewDelegate
	{
	public:
	    
	    virtual ~XKPageViewDelegate(){};
	    XKPageViewDelegate(){};
	    virtual Size sizeForPerPage() = 0;
	    virtual void pageViewDidScroll(XKPageView *pageView){};
	};
	
	class XKPageView:public ScrollView
	{
	public:
	    static XKPageView *create(Size size,XKPageViewDelegate *delegate);
	    virtual bool init(Size size,XKPageViewDelegate *delegate);
	public:
	    void setPageSize(Size size);
	    virtual void setContentOffsetInDuration(Vec2 offset, float dt);
	    virtual void setContentOffset(Vec2 offset);
	private:
	    void performedAnimatedScroll(float dt);
	    int current_index;
	    float current_offset;
	    //调整offset 的函数
	    void adjust(float offset);
	    Size pageSize;
	    CC_SYNTHESIZE(XKPageViewDelegate *, _delegate, Delegate);
	public:
	    int pageCount;
	    void addPage(Node *node);
	    Node *getPageAtIndex(int index);
	};
	
	#endif /* defined(__XKPageView__XKPageView__) */

XKPageView.cpp

	//
	//  XKPageView.cpp
	//  XKPageView
	//
	//  Created by Joueu on 14-11-26.
	//
	//
	
	#include "XKPageView.h"
	
	#define XKPAGEVIEW_TAG 10086
	
	XKPageView *XKPageView::create(Size size,XKPageViewDelegate *delegate)
	{
	    XKPageView *page = new XKPageView();
	    if (page && page -> init(size,delegate)) {
	        page -> autorelease();
	    }else {
	        CC_SAFE_RELEASE(page);
	    }
	    return  page;
	}
	
	bool XKPageView::init(Size size,XKPageViewDelegate *delegate)
	{
	    if (!ScrollView::initWithViewSize(size)) {
	        return false;
	    }
	    //必须有delegate，否则断掉
	    CCASSERT(delegate, "delegate should not be NULL!");
	    setDelegate(delegate);
	    if (_delegate) {
	        //获取page的大小
	        pageSize = _delegate -> sizeForPerPage();
	    }
	    //init Data
	    pageCount = 0;
	    current_index = 0;
    
    this -> setTouchEnabled(false);
    
    auto listener = EventListenerTouchOneByOne::create();
    listener -> onTouchBegan = [&](Touch *touch, Event *event){
        _dragging = false;
        if (_direction == ScrollView::Direction::HORIZONTAL) {
            current_offset = this -> getContentOffset().x;
        }else {
            current_offset = this -> getContentOffset().y;
        }
        return true;
    };
    listener -> onTouchMoved = [&](Touch *touch, Event *event){
        float start, end;
        if (_direction == ScrollView::Direction::HORIZONTAL) {
            start = touch -> getStartLocation().x;
            end = touch -> getLocation().x;
        }else {
            start = touch -> getStartLocation().y;
            end = touch -> getLocation().y;
        }
        float offset = end - start;
        // * 2的作用是调节滚动速度，需要调滑动速度的 可以改这个值
        if (_direction == ScrollView::Direction::HORIZONTAL)
            this -> setContentOffset(Vec2(current_offset + offset * 2, 0));
        else
            this -> setContentOffset(Vec2(0, current_offset + offset * 2));
    };
    listener -> onTouchEnded = [&](Touch *touch, Event *event){
        float start = current_offset, end;
        if (_direction == ScrollView::Direction::HORIZONTAL) {
            end = this -> getContentOffset().x;
        }else {
            end = this -> getContentOffset().y;
        }
        float offset = end - start;
        this -> adjust(offset);
        _dragging = true;
    };
    Director::getInstance() -> getEventDispatcher() -> addEventListenerWithSceneGraphPriority(listener, this);
    return true;
	}


	void XKPageView::adjust(float offset)
	{
	    Vec2 vec;
	    float xOrY;
	    if (_direction == ScrollView::Direction::HORIZONTAL) {
	        vec = Vec2(-( current_index * (pageSize.width)),0);
	        xOrY = pageSize.width;
	    }else {
	        vec = Vec2(0, -( current_index * (pageSize.height)));
	        xOrY = pageSize.height;
	    }
	    //小于50回到原来位置
	    if  (abs(offset) < 50){
	        this -> setContentOffsetInDuration(vec,0.1f);
	        return;
	    }
	    
	    int i = abs(offset / (xOrY)) + 1;
	    if (offset < 0) {
	        current_index += i;
	    }else {
	        current_index -= i;
	    }
	    
	    if (current_index < 0) {
	        current_index = 0;
	    }else if(current_index > 10){
	        current_index = 10;
	    }
	    
	    if (_direction == ScrollView::Direction::HORIZONTAL) {
	        vec = Vec2(-( current_index * (pageSize.width)),0);
	    }else {
	        vec = Vec2(0, -( current_index * (pageSize.height)));
	    }
	    
	    this -> setContentOffsetInDuration(vec, 0.15f);
	}
	
	void XKPageView::setContentOffset(Vec2 offset)
	{
	    ScrollView::setContentOffset(offset);
	    if (_delegate != nullptr)
	    {
	        _delegate -> pageViewDidScroll(this);
	    }
	}
	
	void XKPageView::setContentOffsetInDuration(Vec2 offset, float dt)
	{
	    ScrollView::setContentOffsetInDuration(offset, dt);
	    this->schedule(CC_SCHEDULE_SELECTOR(XKPageView::performedAnimatedScroll));
	}
	
	void XKPageView::performedAnimatedScroll(float dt)
	{
	    if (_dragging)
	    {
	        this->unschedule(CC_SCHEDULE_SELECTOR(XKPageView::performedAnimatedScroll));
	        return;
	    }
	    
	    if (_delegate != nullptr)
	    {
	        _delegate -> pageViewDidScroll(this);
	    }
	}
	
	void XKPageView::addPage(Node *node)
	{
	    if (_direction == ScrollView::Direction::HORIZONTAL) {
	        node -> setPosition(Point(pageCount * pageSize.width + node -> getPositionX(),node -> getPositionY()));
	        this -> setContentSize(Size((pageCount + 1) * pageSize.width,pageSize.height));
	    }else {
	        node -> setPosition(Point(node -> getPositionX(),pageCount * pageSize.height + node -> getPositionY()));
	        this -> setContentSize(Size(pageSize.width,(pageCount + 1) *pageSize.height));
	    }
	    node -> setTag(pageCount + XKPAGEVIEW_TAG);
	    _container -> addChild(node);
	    pageCount ++;
	    
	}
	
	Node *XKPageView::getPageAtIndex(int index)
	{
	    if (index < pageCount && index >= 0) {
	        return _container -> getChildByTag(index + XKPAGEVIEW_TAG);
	    }
	    return  NULL;
	}
	
测试用的HelloWorld.h

	#ifndef __HELLOWORLD_SCENE_H__
	#define __HELLOWORLD_SCENE_H__	
	#include "cocos2d.h"
	#include "cocos-ext.h"
	#include "XKPageView.h"
	
	USING_NS_CC_EXT;
	
	class HelloWorld : public cocos2d::Layer, public XKPageViewDelegate
	{
	public:
	    // there's no 'id' in cpp, so we recommend returning the class instance pointer
	    static cocos2d::Scene* createScene();
	
	    // Here's a difference. Method 'init' in cocos2d-x returns bool, instead of returning 'id' in cocos2d-iphone
	    virtual bool init();
	    
	    // implement the "static create()" method manually
	    CREATE_FUNC(HelloWorld);
	    
	    Layer *getContainer();
	    
	    // PageViewDelegate
	    virtual Size sizeForPerPage();
	    virtual void pageViewDidScroll(XKPageView *pageView);
	private:
	    XKPageView *pageView;
	    void addPages();
	};
	
	#endif // __HELLOWORLD_SCENE_H__
	
HelloWorld.cpp
		
	#include "HelloWorldScene.h"
	USING_NS_CC;
	
	#define COIN_WIDTH 212
	#define COIN_GAP 30
	#define COIN_COUNT 11
	
	Scene* HelloWorld::createScene()
	{
	    // 'scene' is an autorelease object
	    auto scene = Scene::create();
	    
	    // 'layer' is an autorelease object
	    auto layer = HelloWorld::create();
	
	    // add layer as a child to scene
	    scene->addChild(layer);
	
	    // return the scene
	    return scene;
	}
	
	// on "init" you need to initialize your instance
	bool HelloWorld::init()
	{
	    //////////////////////////////
	    // 1. super init first
	    if ( !Layer::init() )
	    {
	        return false;
	    }
	    Size visibleSize = Director::getInstance() -> getVisibleSize();
	    //XKPageView::create(可视范围, XKPageViewDelegate, container);
	//    auto page = XKPageView::create(Size(visibleSize.width ,COIN_WIDTH), this, this -> getContainer());
	    pageView = XKPageView::create(Size(visibleSize.width, COIN_WIDTH), this);
	    pageView -> setDirection(ScrollView::Direction::HORIZONTAL);
	    //XKPageView的滚动区域
	//    page -> setContentSize(Size((COIN_WIDTH + COIN_GAP)* COIN_COUNT ,COIN_WIDTH));
	    //container 定位在屏幕中间
	    pageView -> setPosition(Point((visibleSize.width - COIN_WIDTH) * 0.5, (visibleSize.height - COIN_WIDTH) * 0.5));
	    
	    addPages();
	    //设置裁切为false, 这样layer 溢出pageView的Size还能显示，只是为了演示效果而已~
	    pageView -> setClippingToBounds(false);
	    this -> addChild(pageView);
	    
	    /*测试功能而已*/
	    Node *node = pageView -> getPageAtIndex(5);
	    log("tag = %d",node -> getTag());
	    node -> setScale(1.3f);
	    
	    return true;
	}
	
	void HelloWorld::addPages()
	{
	    Size coinSize = Sprite::create("coin.png") -> getContentSize();
	    //11个layer 加到layer 上
	    for (int i = 0; i < COIN_COUNT; i++) {
	        auto sprite = Sprite::create("coin.png");
	        sprite -> setPosition(coinSize.width * 0.5, coinSize.height * 0.5);
	        std::string str = StringUtils::format("%d", i);
	        Label *label = Label::createWithSystemFont(str, "Arial", 60);
	        label -> setTextColor(Color4B(0,0,0,255));
	        Size size = sprite -> getContentSize();
	        label -> setPosition(size.width * 0.5, size.height * 0.5);
	        sprite -> addChild(label);
	        pageView -> addPage(sprite);
	    }
	}
	
	Layer *HelloWorld::getContainer()
	{
	    auto layer = Layer::create();
	    Size coinSize = Sprite::create("coin.png") -> getContentSize();
	    //11个sprite 加到_container 上
	    for (int i = 0; i < COIN_COUNT; i++) {
	        auto sprite = Sprite::create("coin.png");
	        sprite -> setPosition(coinSize.width * 0.5 + i * (coinSize.width + COIN_GAP) , coinSize.height * 0.5);
	        layer -> addChild(sprite);
	        std::string str = StringUtils::format("%d", i);
	        Label *label = Label::createWithSystemFont(str, "Arial", 60);
	        label -> setTextColor(Color4B(0,0,0,255));
	        Size size = sprite -> getContentSize();
	        label -> setPosition(size.width * 0.5, size.height * 0.5);
	        sprite -> addChild(label);
	    }
	    return layer;
	}
	
	Size HelloWorld::sizeForPerPage()
	{
	    //Delegate 的东西，返回每个Page 的Size
	    return Size(COIN_WIDTH + COIN_GAP, COIN_WIDTH);
	}
	
	void HelloWorld::pageViewDidScroll(XKPageView *pageView)
	{
	    //监听滚动时间，可以再这里写滚动时候要添加的代码,比如缩放~
	    log("pageViewDidScroll");
	}
	
>```我把资源都放在CSDN上面了，需要的可以去下载~  ```   
>[http://download.csdn.net/detail/joueu/8202549][3]

[1]:http://helkyle.tk/cocos-2dx-3xPageView/
[2]:http://blog.csdn.net/joueu/article/details/41532763
[3]:http://download.csdn.net/detail/joueu/8202549
