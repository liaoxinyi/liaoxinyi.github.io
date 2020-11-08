---
layout:     post
title:      "Excel不仅仅是表格"
subtitle:   "日常用到的VBA命令记录"
date:       2020-11-08
author:     "ThreeJin"
header-mask: 0.5
catalog: true
header-img: "https://gitee.com/liaoxinyiqiqi/my-blog-images/raw/master/img/wallhaven-481p92.jpg"
tags:
    - VBA
---
> 来源于平时使用过程中的实践和积累，不定期更新

### 前言
如果说是什么让我喜欢上了编程，VB算得上是第一个引领我迈进编程大门的语言。大学的时候，专业要求的必须掌握的基础便是VB，再后来接触到了VBA之后，瞬间觉得打开了新世界的大门。由于工科上经常会使用Excel进行数据处理，比如在石油行业中往往会涉及到很多大量的表格用来进行数据记录，而且数量级还不小。如果还是停留在复制粘贴，那效率简直感人。稍微有点进阶的，vlookup、if函数等等，但是最终让我玩得高兴的，还是VBA（Visual Basical Application）

### 单元格操作
##### 该表数据最大行列数（中间有缺失行列也会纳入计数）
```vb
'统计当前表格第1行最大列数，其中cells所属表格可以根据实际情况改变
Count = Application.ThisWorkbook.ActiveSheet.Cells(1, Columns.Count).End(xlToLeft).Column
'统计当前表格第4列最大行数
Count = Application.ThisWorkbook.ActiveSheet.Cells(Rows.Count, 4).End(xlUp).Row
```
#####  选择并删除
```vb
'选择并删除
'方法一：Columns函数
Columns("K:K").Select
Selection.Delete Shift:=xlToLeft   '删除列

'方法二：万能的Range函数
Range("L:L,N:N,P:P").Select
Selection.Delete Shift:=xlToLeft

Range("24:24,26:26,28:28,30:30").Select
Selection.Delete Shift:=xlUp    '删除行

'方法三：Rows函数
Rows("17:24").Select
Selection.Delete Shift:=xlUp

'如果只想清除的话，可以使用ClearContents方法，下面这个是示意清除本sheet中全部内容
Application.ThisWorkbook.ActiveSheet.Cells.ClearContents
Columns("D:E").ClearContents
Range(Cells(2, 24), Cells(5000, 5000)).ClearContents
```
##### 字符处理
```vb
'时间日期字符-方法一
If VBA.IsDate(mySheet.Cells(4, 1)) = False Then
  Year = LTrim(VBA.Left(Str(mySheet.Cells(j, 1)), 5))
  Month = LTrim(VBA.Right(Str(mySheet.Cells(j, 1)), 2))
  mySheet.Cells(j, 1) = Year & "/" & Month & "/" & "1"
  mySheet.Cells(j, 1).NumberFormat = "yyyy/m/d"
End If
'时间日期字符-方法二
If VBA.IsDate(mySheet.Cells(j, dateIndex)) = True Then
  Day = Str(Format(mySheet.Cells(j, dateIndex), "dd"))
  Month = Str(Format(mySheet.Cells(j, dateIndex), "mm"))
  Year = Str(Format(mySheet.Cells(j, dateIndex), "yyyy"))
Else
  Year = LTrim(VBA.Left(Str(mySheet.Cells(j, dateIndex)), 5))
  Month = LTrim(VBA.Mid(Str(mySheet.Cells(j, dateIndex)), 6, 2))
  Day = LTrim(VBA.Right(Str(mySheet.Cells(j, dateIndex)), 2))
End If
'去除字符中的空格
Day = Application.WorksheetFunction.Trim(Day)
Month = Application.WorksheetFunction.Trim(Month)
Year = Application.WorksheetFunction.Trim(Year)
```
### 各种交互框
##### inputBox
```vb
    Dim AddCountsString As Variant
    Dim IntervalNums As Integer
    AddCountsString = InputBox("请输入XXXX", "提示", "1")
    If AddCountsString = "" Then
        End
    Else
        IntervalNums = CInt(AddCountsString)
    End If 
```
##### 查找特定文件个数
```vb
FileCount = 0
FileName = Dir(FilePath & "\*.xls") '依次找寻指定路径中的*.xls文件,返回值是不包含路径的单个文件名字
    Do While FileName <> "" '当指定路径中有文件时进行循环
        FileCount = FileCount + 1
	XXXXXX
	XXXXXX
	XXXXXX
        FileName = Dir '找寻下一个*.xls文件
    Loop
```
##### 批量画图设置
```vb
Sub singleChart(myExcel As Excel.Application, myBook As Excel.Workbook, mySheet As Excel.Worksheet, myChart As ChartObject, startRows As Integer, endRows As Integer)
    '参数说明：startRows代表起始行数，endRows是终止行数
    Dim plotLeft As Integer, plotTop As Integer, plotWidth As Integer, plotHeight As Integer
    Dim xColums As Integer, yColums As Integer, seriesName As Variant

    '设置图的属性
    plotLeft = 100:   plotTop = 100:    plotWidth = 600:    plotHeight = 200
    Set myChart = mySheet.ChartObjects.Add(plotLeft, plotTop, plotWidth, plotHeight)
    With myChart.Chart
        '图表类型
        .charttype = xlXYScatterLinesNoMarkers
        '数据源(横坐标数据、纵坐标数据、曲线名称)
        xColums = 1:    yColums = 7:    seriesName = mySheet.Cells(2, 7)
        .SeriesCollection.NewSeries
        .SeriesCollection(1).XValues = mySheet.Range(mySheet.Cells(startRows, xColums), mySheet.Cells(endRows, xColums))
        .SeriesCollection(1).Values = mySheet.Range(mySheet.Cells(startRows, yColums), mySheet.Cells(endRows, yColums))
        .SeriesCollection(1).Name = seriesName
        .Axes(xlValue).MajorGridlines.Delete    '删除网格
        .DisplayBlanksAs = xlZero   '空值连线
        
        '设置图例位置
        With .Legend
            For i = 1 To myChart.Chart.SeriesCollection.Count
                With .LegendEntries(i).Font
                    .Name = "黑体"
                    .FontStyle = "加粗"
                    .Bold = True
                    .Size = 10.5
                    .Strikethrough = False
                    .Superscript = False
                    .Subscript = False
                    .OutlineFont = False
                    .Shadow = False
                    .Underline = xlUnderlineStyleNone
                    .ColorIndex = 1
                    .Background = xlTransparent
                End With
            Next i
            .Left = 150
            .Top = 6
            .Height = 20
            .Width = 300
        End With
        
        '设置绘图区大小
        .PlotArea.Width = 500
'        .PlotArea.Left = 50
        .PlotArea.InsideLeft = 80

        '设置某条曲线的坐标轴个数
        .SeriesCollection(1).AxisGroup = 1

        '设置坐标轴极值
    '    .Axes(xlCategory).MinimumScaleIsAuto = True
        .Axes(xlCategory).MinimumScale = CLng(mySheet.Cells(startRows, xColums))
        .Axes(xlCategory).MaximumScale = CLng(mySheet.Cells(endRows, xColums))
'        .Axes(xlCategory).MajorUnit = 705
        .Axes(xlValue).MinimumScale = 0
        
        '设置第二纵坐标极值(如果有必要)
'        .Axes(xlValue, xlSecondary).MinimumScale = 0
'        .Axes(xlValue, xlSecondary).MaximumScale = 100

        '设置主横纵坐标轴名字
        .Axes(xlCategory).HasTitle = True
        .Axes(xlCategory).AxisTitle.Text = "XXX"
        .Axes(xlValue).HasTitle = True
        .Axes(xlValue).AxisTitle.Text = "XXX"
        '主次纵坐标轴名字(如果有必要)
'        .Axes(xlValue, xlSecondary).HasTitle = True
'        .Axes(xlValue, xlSecondary).AxisTitle.Text = "月产气,10^4m^3"

        '设置坐标轴格式(字体和线条粗细)
        With .Axes(XXX)  'xlValue或者xlCategory或者(xlValue, xlSecondary)
            '设置字体,如果不需要显示横坐标标签则可以设置.TickLabelPosition = xlNone
            .TickMarkSpacing = 5
            .TickLabelSpacing = 10
            .TickLabels.NumberFormatLocal = "yyyy.m"        '"0.E+00"等等格式都可以
            With .TickLabels.Font
                .Name = "Times New Roman"
                .FontStyle = "常规"
                .Size = 10
                .Strikethrough = False
                .Superscript = False
                .Subscript = False
                .OutlineFont = False
                .Shadow = False
                .Underline = xlUnderlineStyleNone
                .ColorIndex = 1
                .Background = xlTransparent
            End With
            '线条粗细
            With .Format.Line
                .Visible = msoTrue
                .Weight = 1.5
                .ForeColor.RGB = RGB(0, 0, 0)
                .Transparency = 0
            End With
            .MajorTickMark = xlInside
            '次刻度线
            'Selection.MinorTickMark = xlInside
            'Selection.MinorTickMark = xlNone
        End With

        '设置曲线颜色和粗细
        With .SeriesCollection(XXX).Format.Line
            .Transparency = 0
            .Visible = msoTrue
            .ForeColor.RGB = RGB(255, 0, 255)
            .Weight = 0.5
        End With
    End With
End Sub
```
##### 进度条+时间表
```vb
tt = Timer
With FormJinDuTiao
    .Show 0   '//显示窗体

    For j = 1 To FileNum
        .Label1.Width = Int(j / FileNum * 324) '//窗体设置：底色为红色，宽度自动代码设置
        .Frame1.Caption = CStr(Round((j / FileNum * 100), 0)) & "%" '//显示内容，窗体设置：长度设为：324
        .Caption = "画图中，请勿关闭进度条"         '//根据实际情况改一下显示内容"
        DoEvents
	XXXXXXXXX
	XXXXXXXXX
End With
Unload FormJinDuTiao  '//关闭窗体
TimeMinute = VBA.Int((Timer - tt) / 60)
TimeSecond = (Timer - tt) Mod 60
MsgBox ("统计完毕,用时" & TimeMinute & "分" & TimeSecond & "秒")

```
##### 表的打开和关闭
```vb
Dim xlAppOut As Excel.Application
Dim xlBookOut As Excel.Workbook
Dim xlSheetOut As Excel.Worksheet

SaveFilePath = ThisWorkbook.Path & "\结果" & Format(Date, "yyyy年m月d日") & Format(Time, "hh时mm分") & ".xlsx"
Set xlAppOut = New Excel.Application
xlAppOut.SheetsInNewWorkbook = 1
Set xlBookOut = xlAppOut.Workbooks.Add
xlBookOut.SaveAs SaveFilePath
'关闭该项就会在后台进行操作，屏幕上不会有任何显示
xlAppOut.Visible = False

For i = 1 To XXX
    xlBookOut.Sheets.Add after:=xlBookOut.Sheets(i)
Next i

Set xlSheetOut = xlBookOut.Sheets(XXX)


Dim xlAppIn As Excel.Application
Dim xlBookIn As Excel.Workbook
Dim xlSheetIn As Excel.Worksheet

Set xlAppIn = New Excel.Application
Set xlBookIn = xlAppIn.Workbooks.Open(XXXXXX)
Set xlSheetIn = xlBookIn.Sheets(XXX)

XXXXXX
XXXXXX
XXXXXX


Set xlSheetIn = Nothing
xlBookIn.Close True
xlAppIn.Quit

Set xlSheetOut = Nothing
xlBookOut.Close True
xlAppOut.Quit
```
##### 选择对话框
```vb
Dim DLGopen2 As FileDialog
Dim TargetName As String
Set DLGopen2 = Application.FileDialog(msoFileDialogFilePicker)
With DLGopen2
        .AllowMultiSelect = False
        .Filters.Add "Excel文件", "*.*", 1
        .InitialFileName = ThisWorkbook.Path & "\2.导出数据\"
        .InitialView = msoFileDialogViewDetails
        .Title = "请选择需要作图的工作簿"
        If .Show = 0 Then
            MsgBox "未选择文件"
            Exit Sub
        Else
            TargetName = .SelectedItems(1)
        End If
End With
```
### 石油常用
```vb
'----------------------------------------------------------------------------------------------------------------------------偏差因子
Sub Zcal(Px, Tx, Ppc, Tpc, Z)
    Ppr = Px / Ppc
    tr = (Tx) / Tpc
    A1 = 0.3265
    A2 = -1.07
    A3 = -0.5339
    A4 = 0.01569
    A5 = -0.05165
    A6 = 0.5475
    A7 = -0.7361
    a8 = 0.1844
    a9 = 0.1056
    a10 = 0.6134
    a11 = 0.721
    yp = Ppr / (tr - 0.6903)
    Count = 1
    
    If yp > 5.56 Then
        o = 0.9872791 * Ppr ^ 0.3022868 / 10 ^ (0.5491962 * tr)
    Else
        o = 0.9420554 * Ppr ^ 1.071221 / 10 ^ (0.7627628 * tr)
    End If
    
    Do
        X = -0.27 * Ppr / tr + o + (A1 + A2 / tr + A3 / tr ^ 3 + A4 / tr ^ 4 + A5 / tr ^ 5) * o ^ 2 + (A6 + A7 / tr + a8 / tr ^ 2) * o ^ 3 - a9 * (A7 / tr + a8 / tr ^ 2) * o ^ 6 + a10 * (1 + a11 * o ^ 2) * (o ^ 3 / tr ^ 3) * Exp(-a11 * o ^ 2)
        X0 = 1 + 2 * (A1 + A2 / tr + A3 / tr ^ 3 + A4 / tr ^ 4 + A5 / tr ^ 5) * o + 3 * (A6 + A7 / tr + a8 / tr ^ 2) * o ^ 2 - 6 * a9 * (A7 / tr + a8 / tr ^ 2) * o ^ 5 + (a10 / tr ^ 3) * (3 * o ^ 2 + a11 * (3 * o ^ 4 - 2 * a11 * o ^ 6)) * Exp(-a11 * o ^ 2)
        o = o - X / X0
        Count = Count + 1
    Loop Until Count > 5
    
        Z = 1 + (A1 + A2 / tr + A3 / tr ^ 3 + A4 / tr ^ 4 + A5 / tr ^ 5) * o + (A6 + A7 / tr + a8 / tr ^ 2) * o ^ 2 - a9 * (A7 / tr + a8 / tr ^ 2) * o ^ 5 + a10 * (1 + a11 * o ^ 2) * (o ^ 2 / tr ^ 3) * Exp(-a11 * o ^ 2)
End Sub
'----------------------------------------------------------------------------------------------------------------------------Bg体积系数
Sub Bgca(Px, Tx, Zx, Bgx)
    Bgx = 3.447 * 10 ^ (-4) * Zx * Tx / Px
End Sub
Sub ρca(Px, Tx, Zx, γ, ρx)
    Pm = Px * 10 ^ 6
    Mg = 28.97 * γ
    r = 8315
    ρx = Mg * Pm / Zx / r / Tx
End Sub
'----------------------------------------------------------------------------------------------------------------------------气体粘度
Sub μgca(Tavg, pavg, Zg, γg, μgx)
   Dim m!, K!, X!, y!
   m = 28.96 * γg
   K = ((22.65 + 0.0388 * m) * Tavg ^ 1.5) / (209.2 + 19.26 * m + 1.8 * Tavg)
   X = 3.448 + 548 / Tavg + 0.01 * m
   y = 2.447 - 0.224 * X
   μgx = 10 ^ (-4) * K * Exp(X * (3.4844 * γg * pavg / Zg / Tavg) ^ y)
End Sub

'----------------------------------------------------------------------------------------------------------------------------气体拟压力
Sub Pmcal(Plow, Pup, T, Ppc, Tpc, γg, DeltaPm)
Dim Number As Double
Dim DeltaP As Double
Dim countP As Double
Dim Pt() As Double

Number = 200
DeltaP = (Pup - Plow) / Number
ReDim Pt(Number) As Double
For i = 0 To Number
    Pt(i) = Plow + DeltaP * i
Next i

countP = 0
For i = 1 To Number
    pa1 = Pt(i - 1)
    Call Zcal(pa1, T, Ppc, Tpc, Z)
    Call μgca(T, pa1, Z, γg, μgx)
    mp1 = 2 * pa1 / Z / μgx
    
    pa2 = Pt(i)
    Call Zcal(pa2, T, Ppc, Tpc, Z)
    Call μgca(T, pa2, Z, γg, μgx)
    mp2 = 2 * pa2 / Z / μgx
    mp3 = (mp1 + mp2) / 2 * DeltaP
    countP = countP + mp3
Next i
    DeltaPm = countP
End Sub

'----------------------------------------------------------------------------------------------------------------------------拟压力反算
Sub PmInverse_cal(Pm, Pup, Temp, Ppc, Tpc, γg, Plow)
Dim Preal As Double
Dim Pmin As Double
Dim Pmax As Double
Dim dP As Double
Dim Pmtemp As Double
Dim ev As Double
Preal = 0
Pmin = 0.1
Pmax = Pup
dP = (Pmax - Pmin) / 2
Preal = Pmin

Do
    Preal = Preal + dP
    Call Pmcal(Preal, Pup, Temp, Ppc, Tpc, γg, Pmtemp)
    If Pmtemp > Pm Then
        Pmin = Preal
        dP = (Pmax - Pmin) / 2
    ElseIf Pmtemp < Pm Then
        Pmax = Preal
        dP = -(Pmax - Pmin) / 2
    End If
    ev = Abs(Pmtemp - Pm)
Loop Until Abs(dP) <= 0.01
    Plow = Preal
End Sub

'----------------------------------------------------------------------------------------------------------------------------P/Z反算
Sub PZInverse_cal(PZcal, Pup, Temp, Ppc, Tpc, Plow)
Dim Preal As Double
Dim Pmin As Double
Dim Pmax As Double
Dim dP As Double
Dim Ztemp As Double
Dim ev As Double
Preal = 0
Pmin = 0.1
Pmax = Pup
dP = (Pmax - Pmin) / 2
Preal = Pmin

Do
    Preal = Preal + dP
    Call Zcal(Preal, Temp, Ppc, Tpc, Ztemp)
    tttt = Preal / Ztemp
    If tttt < PZcal Then
        Pmin = Preal
        dP = (Pmax - Pmin) / 2
    ElseIf tttt > PZcal Then
        Pmax = Preal
        dP = -(Pmax - Pmin) / 2
    End If
    ev = Abs(tttt - PZcal)
Loop Until Abs(dP) <= 0.01
    Plow = Preal
End Sub
```
### 尚未分类的操作
```vb
Sub getDataBase()
    Dim xlAppOut As Excel.Application
    Dim xlBookOut As Excel.Workbook
    Dim xlSheetOut As Excel.Worksheet
    
    SaveFilePath = ThisWorkbook.Path & "\结果" & Format(Date, "yyyy年m月d日") & Format(Time, "hh时mm分") & ".xlsx"
    Set xlAppOut = New Excel.Application
    xlAppOut.SheetsInNewWorkbook = 1
    Set xlBookOut = xlAppOut.Workbooks.Add
    xlBookOut.SaveAs SaveFilePath
    xlAppOut.Visible = False
    
    Set xlSheetOut = xlBookOut.Sheets(1)
    
    
    Dim xlAppIn As Excel.Application
    Dim xlBookIn As Excel.Workbook
    Dim xlSheetIn As Excel.Worksheet
    Dim TargetName As String
    Dim RowsCount As Integer
    
    Call getExcelName(TargetName)
    Set xlAppIn = New Excel.Application
    Set xlBookIn = xlAppIn.Workbooks.Open(TargetName)
    Set xlSheetIn = xlBookIn.Sheets(TargetName)
    RowsCount = xlSheetIn.Cells(xlSheetIn.Rows.Count, 1).End(xlUp).Row
    
    Data1 = xlSheetIn.Range(xlSheetIn.Cells(1, 1), xlSheetIn.Cells(RowsCount, 2))
    Data2 = xlSheetIn.Range(xlSheetIn.Cells(1, 4), xlSheetIn.Cells(RowsCount, 4))
    Data3 = xlSheetIn.Range(xlSheetIn.Cells(1, 7), xlSheetIn.Cells(RowsCount, 11))
    
    xlSheetOut.Range(xlSheetOut.Cells(1, 1), xlSheetOut.Cells(RowsCount, 2)) = Data1
    xlSheetOut.Range(xlSheetOut.Cells(1, 3), xlSheetOut.Cells(RowsCount, 3)) = Data2
    xlSheetOut.Range(xlSheetOut.Cells(1, 4), xlSheetOut.Cells(RowsCount, 8)) = Data3
    
    Set xlSheetIn = Nothing
    xlBookIn.Close True
    xlAppIn.Quit
    
    Set xlSheetOut = Nothing
    xlBookOut.Close True
    xlAppOut.Quit

End Sub

Sub getExcelName(TargetName)
    Dim DLGopen2 As FileDialog
    Set DLGopen2 = Application.FileDialog(msoFileDialogFilePicker)
    With DLGopen2
            .AllowMultiSelect = False
            .Filters.Add "Excel文件", "*.*", 1
            .InitialFileName = ThisWorkbook.Path
            .InitialView = msoFileDialogViewDetails
            .Title = "请选择需要作图的工作簿"
            If .Show = 0 Then
                MsgBox "未选择文件"
                TargetName = ""
                Exit Sub
            Else
                TargetName = .SelectedItems(1)
            End If
    End With
End Sub

Month = Month(XXX)
xlSheetOut.Range(xlSheetOut.Cells(2, 1), xlSheetOut.Cells(ResDataNum, 11)).Columns.AutoFit
xlSheetOut.Range(xlSheetOut.Cells(2, 1), xlSheetOut.Cells(ResDataNum, 11)).HorizontalAlignment = xlCenter
xlSheetOut.Range(xlSheetOut.Cells(1, 1), xlSheetOut.Cells(1, 10)).Merge
```
