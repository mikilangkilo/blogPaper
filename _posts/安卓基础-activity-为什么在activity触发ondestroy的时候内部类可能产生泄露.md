---
title: 安卓基础-为什么在activity触发ondestroy的时候内部类可能产生泄露
date: 2018-10-21 18:41:36
tag : "activity"
category : "安卓基础"
description : "为什么在activity触发ondestroy的时候内部类可能产生泄露"
---
基于内部类的泄露是Android泄露的一个很常见的问题。

这方面自从使用了rxjava之后有很多好转，凡是内部类的我都会转换成为observable然后bindtolifecycle，这样通过生命周期的强制绑定，能够一定程度上面减少泄露发生的问题。

在lifecycle中，一般如果不指明的话，在start时订阅的事件将会在stop的时候dispose，在resume时订阅的事件将会在pause的时候dispose。

```
switch (lastEvent) {
                case CREATE:
                    return ActivityEvent.DESTROY;
                case START:
                    return ActivityEvent.STOP;
                case RESUME:
                    return ActivityEvent.PAUSE;
                case PAUSE:
                    return ActivityEvent.STOP;
                case STOP:
                    return ActivityEvent.DESTROY;
                case DESTROY:
                    throw new OutsideLifecycleException("Cannot bind to Activity lifecycle when outside of it.");
                default:
                    throw new UnsupportedOperationException("Binding to " + lastEvent + " not yet implemented");
            }
```

生命周期就是上面这种，lifecycle在必要的地方做了很详细的处理，通过lifecyclesubject来根据绑定的生命周期来不断的发送接下来的activityevent（生命周期事件）。直到当发送下来一个取消的通知，之后就会取消订阅。

lifecycle的确很好用，但是没有办法所有的地方都使用到rxjava。因此关于内部类的泄露还是要明确讨论一下。


# 内部类泄露


回到内部类泄露的地方。Android的内部类分为静态内部类和非静态内部类，内部类又分一些匿名的，成员的等等。非静态内部类会潜在的持有外部类的引用，因此当有耗时操作的时候，就会导致外部类的泄露。

activity在ondestroy的时候，凡是引用到他的内部类，假如没有结束的话，就会导致activity被泄露，因为activity是被强引用。

解决的方法其实看来很简单，1.在activity结束的时候强制释放掉2.将对activity的引用改为弱引用。

# 引用细分

引用只有四种：强引用（内存泄露罪魁祸首），软引用，弱引用，虚引用。

一般将强引用转化为弱引用即可避免强制引用带来的锅。


# loginChecker优化

代码中为了方便做登录的检查，同时为了防止包过大，没有使用切片，而是写了一个loginchecker辅助类

```
package com.haomaiyi.base.util;

import Android.annotation.SuppressLint;
import Android.app.Activity;
import Android.text.TextUtils;

import com.haomaiyi.base.updatemanager.Logger;
import com.haomaiyi.fittingroom.AppApplication;
import com.haomaiyi.fittingroom.domain.interactor.account.GetCurrentAccount;
import com.haomaiyi.fittingroom.domain.model.account.Account;
import com.haomaiyi.fittingroom.domain.model.account.AnonymousAccount;
import com.haomaiyi.fittingroom.ui.LoginActivity;
import com.orhanobut.hawk.Hawk;
import com.trello.rxlifecycle2.LifecycleProvider;

import javax.inject.Inject;

import io.reactivex.Observable;
import io.reactivex.Android.schedulers.AndroidSchedulers;
import io.reactivex.schedulers.Schedulers;

@SuppressLint("CheckResult")
public class LoginChecker {

    private static LoginChecker instance;
    public LoginProceed proceed;
    @Inject
    GetCurrentAccount getCurrentAccount;
    //    private boolean loginStat = false;
    private int uid = -1;
    private boolean logSuccess = false;

    public LoginChecker() {
        AppApplication.getInstance().getUserComponent().inject(this);
    }

    public static LoginChecker getInstance() {
        if (instance == null) {
            synchronized (LoginChecker.class) {
                if (instance == null) {
                    instance = new LoginChecker();
                }
            }
        }
        return instance;
    }

    public void onLoginStart() {
        logSuccess = false;
    }

    public void onLoginFinish() {
        // TODO: 2018/9/30
    }

    public LgnStat getLoginStat() {
        Account ac = getCurrentAccount.executeSync();
        LgnStat stat;
        if (ac instanceof AnonymousAccount) {
            stat = LgnStat.ANONYMOUS;
        } else if (TextUtils.isEmpty(ac.getPhonenumber())) {
            stat = LgnStat.NEED_NUMBER;
        } else {
            stat = LgnStat.COMPLETE;
        }
        return stat;
    }

    public int getUid() {
        uid = getCurrentAccount.executeSync().getId();
        return uid;
    }

    public void setUid(int uid) {
        this.uid = uid;
    }

    public void check(LifecycleProvider lifecycle, LoginCallback callback) {

        final Observable<Account> ob = getCurrentAccount.getObservable()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread());

        if (lifecycle != null)
            ob.compose(lifecycle.bindToLifecycle());

        ob.subscribe(ac -> {
            uid = ac.getId();
            LgnStat stat;
            if (ac instanceof AnonymousAccount) {
                stat = LgnStat.ANONYMOUS;
            } else if (TextUtils.isEmpty(ac.getPhonenumber())) {
                stat = LgnStat.NEED_NUMBER;
            } else {
                stat = LgnStat.COMPLETE;
            }
            callback.onLogin(stat);
        });
    }

    public void loginAndproceed(Activity src, LifecycleProvider lifecycleProvider, LoginProceed proceed) {
        Logger.i("check=" + this);
        check(lifecycleProvider, new LoginCallback() {
            @Override
            public void onLogin(LgnStat stat) {
                switch (stat) {
                    case ANONYMOUS:
                        LoginChecker.this.proceed = proceed;
                        LoginActivity.start(src);
                        break;
                    case NEED_NUMBER:
                        LoginChecker.this.proceed = proceed;
                        LoginActivity.start(src, LgnStat.NEED_NUMBER);
                        break;
                    case COMPLETE:
                        proceed.onProceed();
                        break;
                }
            }
        });
    }

    /**
     * Must be called after cancel login behavior.
     */
    public void clear() {
        proceed = null;
    }

    /**
     * Must be called after login success.
     */
    public boolean consume(int id, LoginActivity act) {
        uid = id;
        Hawk.put("SHOW_GIFT", true);
        if (proceed == null)
            return false;
        proceed.onLoginAndProceed(id, act);
        proceed = null;
        return true;
    }

    public enum LgnStat {
        // 匿名用户
        ANONYMOUS,
        // 有手机号
        COMPLETE,
        // 无手机号
        NEED_NUMBER
    }

    public interface LoginProceed {
        void onProceed();

        void onLoginAndProceed(int id, LoginActivity act);
    }

    public interface LoginCallback {
        void onLogin(LgnStat loginStat);
    }
}
```

这个类虽然不是内部类，但是其携带的内部类以接口形式获取了activity，以activity作为实例，做了一些别的事情。而且有个最重要的问题，就是启动loginactivity的时候使用的是获取的实例。同时整个类又绑定了lifecycle。这就导致了一个很有趣的问题，就是当使用实例来启动的时候触发了onpause和onstop，此时刚好gc，导致无法回收。

因此此时需要更改几个地方，一是不在使用原来传过来的activity，改用application的context，这样就不会再出现强制捆绑的地方。其次是将该类中使用到的任何引用的地方改成weakreference。atu