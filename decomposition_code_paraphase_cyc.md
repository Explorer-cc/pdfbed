![image-20230129234945392](https://fig-lianxh.oss-cn-shenzhen.aliyuncs.com/image-20230129234945392.png)

```stata
bys ind2 year: egen outputind2=total(qt_zjz2)
bys year: egen outputyear=total(qt_zjz2)
gen weightind2=outputind2/outputyear    //weightind2为行业zjz占该年总增加值的比例作为权重
gen weightfirm=qt_zjz2/outputind2   //weightfirm为企业zjz占行业增加值的比例作为权重
//计算wj,t  wj,t-1等指标

preserve
duplicates drop ind2 year, force
xtset ind2 year


****按照简单分解公式_计算四个差分乘积再进行求和
gen Diff_water=d.wateryear0

gen btw1=weightind2*d.waterind2
//gen wtn1=d.weightind2*l.waterind2
gen btw2=l.weightind2*d.waterind2
//gen wtn2=d.weightind2*waterind2

foreach v of varlist btw1 btw2 {
bys year: egen Dcmp_`v'=total(`v')
}
//根据分解公式 Diff_water=Dcmp1+Dcmp2
//转化为比例
gen btw1share=Dcmp_btw1/Diff_water*100
gen btw2share=Dcmp_btw2/Diff_water*100

duplicates drop year, force
keep year Diff_water Dcmp_*
/* 图5 */
tw connected Dcmp_btw1 Dcmp_btw2 year if year<2015
graph save "G:\CO2\水\table\figure5.dta", replace
restore
```



OP分解公式

![image-20230129235248528](https://fig-lianxh.oss-cn-shenzhen.aliyuncs.com/image-20230129235248528.png)

![image-20230129235332183](https://fig-lianxh.oss-cn-shenzhen.aliyuncs.com/image-20230129235332183.png)

```stata
*** 2.5 行业内进一步分解 ***
**当前数据的层次结构year-ind2-firmtype-type 
/* 分解 */
forvalues i=2007/2014{  //按年份操作
preserve
keep if year==`i'|year==`i'+1    //只研究先后两年

bys panelid: gen Total_Firm=_N    //计算企业存在的year年份数量
gen type=0 if Total_Firm==2   //Type0指连续两年均在市场中存活
replace type=1 if Total_Firm==1&year==`i'    //Type1指今年存活第二年退出市场
replace type=2 if Total_Firm==1&year==`i'+1   //Type2指今年不存活明年进入市场

**由于weightfirm为企业在行业的权重，需要bys ind2 year
bys ind2 year: egen water_weight=total(water*weightfirm)  //water_weight行业水消费量加权和
bys ind2 year type: egen watermean=mean(water)    //watermean行业中不同类型企业的水消费量的均值

bys ind2 year type: egen outputtype=total(qt_zjz2)   //outputtype行业的总GDP产值
gen weighttype=outputtype/outputind2   //weighttype为一个行业中GDP产值这一指标各个类型/行业的比例权重
bys ind2 year type: egen watertype=total(water*weightfirm) //watertype行业中各个类型企业的水消费量加权和
replace watertype=watertype/weighttype  //watertype重新定义为一个行业中各个企业按GDP比例折算的平均水消费量
//他这个除比例我感觉巨奇怪... ??

duplicates drop ind2 year type, force
egen ind2type=group(ind2 type)
xtset ind2type year

gen Diff_total=d.water_weight   //公式(3)左端ΔΦ
gen Diff_term0=d.watertype   //ΦS2-ΦS1
gen Diff_term1=d.watermean  //Part I 组内效应——水消费量均值的差分
gen Diff_term2=Diff_term0-Diff_term1   //Part II 组间效应——利用公式(3)关系转换求解Part II组间效应

/*
S——存活
E——进入
X——退出 
*/
  
**现在有数据的企业 要么是第一年存活+下一年退出   要么是第二年存活+第二年刚进入
bys ind2 year: egen ex1_add_es1=total(watertype)  
bys ind2 year: egen ee2_add_es2=total(watertype)
gen Diff_term3=weighttype*(ex1_add_es1-watertype*2) if type==1   //退出效应   计算的ex1addes1是存活加上退出，目标是退出效应，所以对于自己而言要多减去一个水消费量  换言之即为S1+X1-2X1简化计算退出效应   但我总觉得细节上还是不太明确...   ???
gen Diff_term4=weighttype*(watertype*2-ee2_add_es2) if type==2
//同理 2E2-(S2+E2)即为进入效应
foreach v of varlist Diff*{
bys ind2: egen D`v'=mean(`v')   //对每个两年的效应求均值
}

keep year ind2 DDiff*
keep if year==`i'+1
save "G:\CO2\水\table\data_dcmp`i'.dta", replace   //保留preserve数据再合并

restore
}

/* 整理 */
preserve
use "G:\CO2\水\table\data_dcmp2007.dta", clear
forvalues i=2008/2014{
append using "G:\CO2\水\table\data_dcmp`i'.dta"
save "G:\CO2\水\table\data_dcmp.dta", replace
}

forvalues i=1/4{
gen S_`i'=DDiff_term`i'/DDiff_total
bys year: egen MS_`i'=median(S_`i')
}
duplicates drop year, force

graph bar MS_1 MS_2 if year<2015, stack over(year) 
graph save "G:\CO2\水\table\figure6.dta", replace

restore

```

