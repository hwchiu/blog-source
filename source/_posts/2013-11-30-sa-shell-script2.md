---
layout: post
title: 'SA - Shell Script(2)'
date: 2013-11-30 17:02
comments: true
categories: 
---
作業二的部分是要寫一個script,真對參數變化然後使用gunplot去進行繪圖

<!--more-->

程式要求要有下列參數

- o : output file name
- t : type
- c : collor
- n : number of point should used

每個參數都要做錯誤檢查，這邊我使用了 `getopts` 來做參數的取得，並且把對應的值都存起來

然後再一個一個判斷是否有錯誤。

再gnuplot的部分，因為要求X軸必須是反向的，即(-10,-9,-8....0),這部分我採用的是利用一個暫存檔來做

先使用`tail`的方式取得最後 **n**筆資料，然後再透過 `awk` 把項目加上負號並且印出
如`tail -r -n $pointNumber ${inputFile:="/tmp/sysmonitor"} | awk '{ print -NR" "$1}' > $tempInput`
後來有同學說可以再gnuplot中可以使用`using`這個方式來辦到這個結果。

gnuplot的部分，先把所有設定檔寫入暫存檔中，然後再直接透過gnuplot去執行該檔案，最後再刪除這些暫存檔。



``` sh
#!/bin/sh

print_usage()
{
	echo "Usage cpuplot [-h] [-o out_file_name] [-t type] [-c color] -n <60-600>"
}


print_help()
{	
	echo "-o set the output file name. (default: out.png)"
	echo "-t set the graph type. (one of ‘filledcurve’, ‘lines’. default: ‘filledcurve’)"
	echo "set graph color. (in hexadecimal form, default: #1E90FF)"
	echo "set the number of point should use. (must be set. should be in range[60-600]"
	echo "Read LOGFILE environment variable. If it is not set, use /tmp/sysmonitor"
}





# Parse the arguments
while getopts "ho:t:c:n:" opt
do
	case "$opt" in
		h)  print_help; 
		    exit 1 
		    ;;
		o) 
			outName=$OPTARG
			;;
		t) 
			graphType=$OPTARG
			;;
		c) 
			graphColor=$OPTARG
			;;
		n) 
			pointNumber=$OPTARG
			;;
		*)
			exit 1
			;;
	esac

done

# check graph type, which must be filledcurve or lines.
if [ "$graphType" ] ; then
	if [ "$graphType" != "filledcurve" ] && [ "$graphType" != "lines" ] ; then
		echo "type should be one of 'filledcurve' and 'lines'."
	fi	
	if [ "$graphType" == "filledcurve" ] ; then
		graphType="filledcurve y1=0"
	fi	
fi


# check graph color, wich must fit #[0-9a-f]{6}
if [ "$graphColor" ] ; then
	tmp=`echo $graphColor | grep '^#[0-9a-f]\{6\}' `
	if [ -z "$tmp" ] ; then
		echo "color format error."
		exit
	fi
fi

# check point number range in 60 ~ 600
if [ -z $pointNumber ] || [ "$pointNumber" -lt 60 ] || [ "$pointNumber" -gt 600 ] ; then
	print_usage
	exit
fi


# check input files's location

inputFile=`printenv LOGFILE`

#generate a reverse data
tempInput="input2"
`tail -r -n $pointNumber ${inputFile:="/tmp/sysmonitor"} | awk '{ print -NR" "$1}' > $tempInput`


#generate a temp plt file 
tempFile="temp.plt"
`touch $tempFile`

echo "set term png" >> $tempFile
echo "set out '${outName:="out.png"}'" >> $tempFile
echo "set title 'CPU Usage'" >> $tempFile
echo "unset key" >> $tempFile
echo "set grid front" >> $tempFile
echo "set xlabel 'time from now(sec)'" >> $tempFile
echo "set ylabel 'CPU Usage(%)'" >> $tempFile
echo "plot [-$pointNumber:0] [0:100] '$tempInput' with ${graphType:="filledcurve y1=0"} linetype rgb '${graphColor:="#1E90FF"}' " >> $tempFile


`gnuplot $tempFile`

if [ -f $tempInput ] ; then
	`rm $tempInput`
fi

if [ -f $tempFile ] ; then
	`rm $tempFile`
fi



```