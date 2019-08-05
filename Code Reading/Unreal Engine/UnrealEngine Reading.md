# 基础

## 类前缀:
+ U: UObject继承过来的，例如UTexture
+ A: AActor继承过来的，例如AGameMode
+ F: 其他的类和结构，例如FName, FVector
+ T：模板，例如TArray,TMap,TQueue
+ I: 接口类，ITransaction
+ E: 枚举, ESelectionMode
+ B: Boolean, bEnabled
+ G: 全局变量, GIsRHIInitialized

## 用宏定义来包裹C++代码
+ UCLASS 来包裹类
+ USTRUCT 包裹结构
+ UFUNCTION 包裹功能
+ UPROPERTY 包裹属性

## UE4代码中使用自己的基础类型
+ 不适用C++中的（char,short,int,long等）取而代之的是:int32,uint32,uint64,TCHAR,ANSICHAR等　　
> 数值类型在NumericLimits.h中声明，可以详细阅读查询
+ 一般的结构数据类型有::FBox,FColor,FGuid,FVariant,FVector,TBigInt,TRange　　

## 容器
+ TArray,TSparseArray-动态数组
+ TLinkedList,TDoubleLinkedList
+ TMap-键值对哈希表
+ TQueue-队列
+ TSet-非有序集

## 智能指针　　
+ TSharedPtr,TSharedRef-一般传统的C++对象　　
+ TWeakPtr-一般传统的C++对象　　
+ TWeakObjPtr-UObject　　
+ TAutoPtr,TScopedPtr　　
+ TUniquePtr

## String 类型
+ FString- 通常的String
+ FText- 本地化，在Slate UI中常使用
+ FName-在UObject中常使用的，String哈希.FName是大小写敏感的

## String文字
+ TEXT()- 创建一个通用的String类型，TEXT("Hello");
+ LOCTEXT()-创建一个本地化文字,LOCTEXT("Namespace","Name","Hello");
+ NSLOCTEXT()-在一个域名空间内的本地化，NSLOCTEXT("Name","Hello");

# Rendering Part Reading Notes

## AtmosphereRendering
