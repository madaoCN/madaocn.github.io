---
layout: post
title: MDCalendar 一个简单的日历demo
date: 2018-11-01
Author: madaoCN
categories: 
tags: [源码分享]
comments: true
---

# MDCalendar

a simple calendar demo as following

![calendar.gif](https://upload-images.jianshu.io/upload_images/1749699-0958b638a2ae294b.gif?imageMogr2/auto-orient/strip)

using collectionview to realize the calendar
```swift
    func collectionView(_ collectionView: UICollectionView, cellForItemAt indexPath: IndexPath) -> UICollectionViewCell {
        
        let cell = collectionView.dequeueReusableCell(withReuseIdentifier: String.init(describing: MADCalendarCell.self), for: indexPath) as! MADCalendarCell
        
        if indexPath.row < self.weekTitles.count {
            cell.titleLabel.text = self.weekTitles[indexPath.row]
        }else {
            let index = indexPath.row - self.weekTitles.count
            if index < self.currentMonthData.dayArr.count {                
                cell.model = self.currentMonthData.dayArr[index]
            }
        }
        return cell
    }
```


it is too simple，so i have no more thing to be written.

[github demo](https://github.com/madaoCN/MDCalendar)