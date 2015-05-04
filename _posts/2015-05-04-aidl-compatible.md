---
layout: post
title:  "Android aidl backward & forward compatible"
date:   2015-05-04
categories: aidl
tags: aidl
---

# Android aidl兼容性 #

当对外接口以aidl方式提供给外部调用的话，就需要考虑接口的向前向后兼容性问题。

这里我们只讨论函数的兼容性。暂不考虑object的兼容性。

涉及到：

- 增加函数
- 删除函数
- 函数修改

# aidl 跨进程调用 #

以一下aidl代码为例
	
	interface IHostLib {
    	String test(String className);
	}

编译完的java代码

	public interface IHostLib extends android.os.IInterface {
	    /** Local-side IPC implementation stub class. */
	    public static abstract class Stub extends android.os.Binder implements com.baidu.gpt.hostdemo.lib.IHostLib {
	        private static final java.lang.String DESCRIPTOR = "com.baidu.gpt.hostdemo.lib.IHostLib";
	
	        /** Construct the stub at attach it to the interface. */
	        public Stub() {
	            this.attachInterface(this, DESCRIPTOR);
	        }
	
	        /**
	         * Cast an IBinder object into an com.baidu.gpt.hostdemo.lib.IHostLib
	         * interface, generating a proxy if needed.
	         */
	        public static com.baidu.gpt.hostdemo.lib.IHostLib asInterface(android.os.IBinder obj) {
	            if ((obj == null)) {
	                return null;
	            }
	            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
	            if (((iin != null) && (iin instanceof com.baidu.gpt.hostdemo.lib.IHostLib))) {
	                return ((com.baidu.gpt.hostdemo.lib.IHostLib) iin);
	            }
	            return new com.baidu.gpt.hostdemo.lib.IHostLib.Stub.Proxy(obj);
	        }
	
	        @Override
	        public android.os.IBinder asBinder() {
	            return this;
	        }
	
	        @Override
	        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags)
	                throws android.os.RemoteException {
	            switch (code) {
	            case INTERFACE_TRANSACTION: {
	                reply.writeString(DESCRIPTOR);
	                return true;
	            }
	            case TRANSACTION_test: {
	                data.enforceInterface(DESCRIPTOR);
	                java.lang.String _arg0;
	                _arg0 = data.readString();
	                java.lang.String _result = this.test(_arg0);
	                reply.writeNoException();
	                reply.writeString(_result);
	                return true;
	            }
	            }
	            return super.onTransact(code, data, reply, flags);
	        }
	
	        private static class Proxy implements com.baidu.gpt.hostdemo.lib.IHostLib {
	            private android.os.IBinder mRemote;
	
	            Proxy(android.os.IBinder remote) {
	                mRemote = remote;
	            }
	
	            @Override
	            public android.os.IBinder asBinder() {
	                return mRemote;
	            }
	
	            public java.lang.String getInterfaceDescriptor() {
	                return DESCRIPTOR;
	            }
	
	            @Override
	            public java.lang.String test(java.lang.String className) throws android.os.RemoteException {
	                android.os.Parcel _data = android.os.Parcel.obtain();
	                android.os.Parcel _reply = android.os.Parcel.obtain();
	                java.lang.String _result;
	                try {
	                    _data.writeInterfaceToken(DESCRIPTOR);
	                    _data.writeString(className);
	                    mRemote.transact(Stub.TRANSACTION_test, _data, _reply, 0);
	                    _reply.readException();
	                    _result = _reply.readString();
	                } finally {
	                    _reply.recycle();
	                    _data.recycle();
	                }
	                return _result;
	            }
	        }
	
	        static final int TRANSACTION_test = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
	    }
	
	    public java.lang.String test(java.lang.String className) throws android.os.RemoteException;
	}


aidl 调用可以认为为 C/S 结构，被调用者为S，调用者为C

aidl实现代码：

	IHostLib.Stub binder = new IHostLib.Stub() {
        
        @Override
        public String test(String className) throws RemoteException {
            return className;
        }
    };
    
    return binder;


调用代码

	IHostLib hostapi = IHostLib.Stub.asInterface(binder);
	String result = hostapi.test("hello");


调用顺序

1. 通过asInterface函数返回 Proxy 类的实例
2. 通过interface 调用到proxy 的 test() 函数。
3. 通过进程间通信，调用到了 Stub的 onTransact() 函数


Proxy 代表了调用方的接口，Stub代表了被调用方的实现。


两者之间比如这个test()函数是根据 

	static final int TRANSACTION_test = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);

这个变量做了对应。相当于一个id，如果新增一个函数的话比如 test2，编译后会是这样。

	static final int TRANSACTION_test = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
	static final int TRANSACTION_test2 = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);

所以一旦你的 aidl 发布出去，ipc调用的话，跟这个函数名字叫什么其实关系不大（你完全可以重命名服务端的函数名，当然没这个必要），只是根据这个id做关联调用。

# aidl文件修改对asInterface 的影响 #

如果对外公布的aidl只包含test函数

后边由于升级，实现端部分增加了一个函数test2，或者删除了一个函数。

这种情况对于 asInterface 的调用没有任何影响，可以正确的返回接口实例。

只是这种随意的修改需要注意，存在兼容性问题。

# aidl函数向后兼容 #

上边提到了，新增函数，会新增一个函数id，并且递增。第一个函数id index 为0。

并且提到了c-s函数调用是根据这个id来进行的。

基于这个原则，aidl函数的修改就要求

- 增加函数只能在最后增加。
- 不能文件列表的中间插入新函数。
- 不能改变函数列表顺序。
- 不能删除函数。

# aidl函数向前兼容 #

向前兼容目前来看比较简单。

基于向后兼容的原则，向前兼容只涉及到一种情况，即，该函数不存在。

这时候直接调用，也不会抛出异常，如果函数有返回值的话，或返回该类型的默认值。

比如 String 类型，调用的话会返回null。

所以向前兼容我们只需要注意函数的返回值做正常逻辑处理即可。


# 总结 #

上边提到的只是最简单的一种处理方式，某些情况下不是最好的方式。

引用Android developer Dianne Hackborn 的话：

> The formally correct thing is to make the functionality be on a new interface that the app explicitly requests.  (The interface can also contain all of the original functionality for simplicity if you want.)  This is basically the COM interface versioning approach.
      
>       
>
The quick and dirty approach is to take advantage of knowing that the current aidl compiler assigns an interface's methods their identifiers in the order they are declared, so you can add new methods to the end without changing the identifiers for the existing ones.
