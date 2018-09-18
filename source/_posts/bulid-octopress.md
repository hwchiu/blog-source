---
title: Build own octopress
keywords: 'octopress, build, setup'
date: '2013-04-06 13:00'
comments: true
tags:
  - System
abbrlink: 48550
description: 本文用來記錄在安裝 octopress 發生的問題以及如何解決

---

# Basic install
參考下列文章來進行基本的安裝
[setup octopress on windoes from zero to 100](http://tech.marsw.tw/blog/2012/11/23/setup-octopress-on-windows-from-zero-to-100)

# Problem Solved
1.下指令 rake setup_github_pages ，噴出錯誤訊息  
	rake aborted!
	You have already activated rake 10.0.2, but your Gemfile requires rake 0.9.2.2.
	Using bundle exec may solve this....
>修改Gemfile 這個檔案，手動改版本

2.rake setup_github_pages，輸入url後噴出找不到的訊息
	rake aborted!
	no such file or directory - git -remote -v 


>先把git的路徑加入倒環境變數PATH中，在使用windows內建的shell來執行相關指令  
>reference [this](http://www.v2ex.com/t/32542)  

# Upload
因為文章中若含有中文，必須要設定環境變數，加上每次上傳都要先產生文章，在更新上去，懶惰的我就寫了一個batch來使用  
batch.bat  
	set LANG=zh_TW.UTF-8
	set LC_ALL=zh_TW.UTF-8
	bundle exec rake generate & bundle exec rake deploy


***

# Category

## Install


增加一個category_list 外掛,新增 plugins/category_list_tag.rb   
``` ruby
	module Jekyll
	  class CategoryListTag < Liquid::Tag
	    def render(context)
	      html = ""
	      categories = context.registers[:site].categories.keys
	      categories.sort.each do |category|
	        posts_in_category = context.registers[:site].categories[category].size
	        category_dir = context.registers[:site].config['category_dir']
	        category_url = File.join(category_dir, category.gsub(/_|\P{Word}/, '-').gsub(/-{2,}/, '-').downcase)
	        html << "<li class='category'><a href='/#{category_url}/'>#{category} (#{posts_in_category})</a></li>\n"
	      end
	      html
	    end
	  end
	end
	
	Liquid::Template.register_tag('category_list', Jekyll::CategoryListTag)

```


增加aside 
修改 source/_includes/asides/category_list.html
加入下列,記得把category那行頭尾改成{}，這邊是因為使用{}的話，我頁面顯示會不如預期，所以為了顯示而修改
``` 

<section>
  <h1>Categories</h1>
  <ul id="categories">
    <% category_list %> //change <> to {}
  </ul>
</section>
>
```

修改 _config.yml,根據自己需求調整

	default_asides: [asides/category_list.html, asides/recent_posts.html]

## Usage
寫新文章的時候，底下會出現categories的標籤，在後面增加其類別即可  
	categories: [System]
如果想要同時增加到多個類別，就用逗號隔開
	categories: [System ， Life]

***
# Comments

1. 先到[Disqus](http://www.disqus.com/)註冊帳號,其中會有個short_name,這個名稱記住下來，等等會用到
  
![](https://lh6.googleusercontent.com/-S5HLwtIbyTs/Uc_EqbGQk9I/AAAAAAAAAjA/GbbQQNBhy0Q/w479-h558-no/disqus.jpg)


2. 修改_config.yml ，這邊就把剛剛紀錄的short_name給設定上去

	disqus_short_name: your_disqus_short_name
	disqus_show_comment_count: true

3. 上傳重新整理看看

***
# Backup Octopress source
```
	git add *  
	git commit -m "message"  
	git push origin source  

```   


# [Reference]
[Setup Octopress on Windows From Zero to 100](http://tech.marsw.tw/blog/2012/11/23/setup-octopress-on-windows-from-zero-to-100/ "Setup Octopress on Windows From Zero to 100")  

[为octopress添加分类(category)列表](http://codemacro.com/2012/07/18/add-category-list-to-octopress/)
