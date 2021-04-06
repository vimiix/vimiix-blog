---
title: "自助购物机交易源码"
date: 2017-04-05 18:57:34
tags:
  - source code
  - Python
categories: Python

---

![](http://vimiix-blog.oss-cn-qingdao.aliyuncs.com/self-service-machine.jpg)

##### 本文是一篇纯源码，功能上实现了自助购物机的基本交易过程。

<!--more-->

###### (由于没有去深研究实际生产环境中自助购物机的逻辑，所以在逻辑上不排除有考虑不周的地方，如果有不完善的地方，请在评论里指出，互相学习。)

```Python
	#coding=utf-8
	
	#Author:Vimiix
	#Time:2017-04-05
	#Lisence:Apache
	
	#流程分析：
	#   1、顾客投币，投币结束
	#   2、遍历商品，展示商品信息
	#   3、顾客购买饮料，账户结算
	#   4、任意时刻按q退出
	
	print('Welcome!')
	
	g_money = 0
	g_drinks_list = [
	    ('Coffee',5),
	    ('Coco',2.5),
	    ('Milk',3),
	    ('Water',1)
	    ]
	
	#遍历商品，打印商品列表
	def show_drink_list():
	    print('*********Drink List*********')
	    print('Index\tDrink\tPrice')
	    print('----------------------------')
	    for index,drink in enumerate(g_drinks_list):
	        print index,'\t',drink[0],'\t',drink[1]
	    print('****************************')
	
	#退款退出
	def end_shop():
	    global g_money
	    if g_money == 0:
	        print('Your balance is 0 yuan.\nByebye')
	        exit()
	    else:
	        print('Return your balance:%d yuan\nByebye'%g_money)
	        exit()
	
	
	#顾客投币，投币结束以后进入选商品阶段，按q退出
	def recharge():
	    global g_money
	    while True:
	        start_money = raw_input('Please input your money(Press "q" to exit):')
	        i_money = 0
	        if start_money == 'q':
	            end_shop()
	        elif start_money.isdigit() and int(start_money) > 0:
	            i_money = int(start_money)
	            g_money += i_money
	            print('You just input %d yuan.Your acount totaly has %d yuan'%(i_money,g_money))
	            if (raw_input('Do you want to continue inputing?(y/n)：') == 'y'):
	                continue
	            else:
	                print('Your acount totaly has %d yuan.'%g_money)
	                break
	        else:
	            print("Sorry,can't recogonize what your input.Try again.")
	            continue
	
	
	#顾客挑选商品，并显示余额
	def pick_drink():
	    global g_money
	    global g_drinks_list
	    while True:
	        customer_choose = raw_input('Input the INDEX of which you want(Press "q" to exit):')
	        if customer_choose == 'q':
	            end_shop()
	        else:
	            i_customer_choose = int(customer_choose)
	            if( 0 <= i_customer_choose  and i_customer_choose < len(g_drinks_list)):
	                if g_money < g_drinks_list[i_customer_choose][1]:
	                    is_recharge = raw_input("Your balance is not enough.Do you want to recharge?(y/n):")
	                    if is_recharge == 'y':
	                        recharge()
	                    else:
	                        end_shop()
	                else:
	                    g_money -= g_drinks_list[i_customer_choose][1] 
	                    print('Your chooes is %s ,which costs %.1f yuan.Your balance is %.1f yuan.'\
	                          %(g_drinks_list[i_customer_choose][0],g_drinks_list[i_customer_choose][1],g_money))
	            else:
	                print("Sorry,can't find it.")
	
	
	if __name__ == '__main__':
	    recharge()
	    show_drink_list()
	    pick_drink()
```