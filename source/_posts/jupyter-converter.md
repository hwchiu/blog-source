---
title: Translate jupyter notebook to python script
keywords: 'Python,Jupyter'
date: 2017-12-25 12:50:29
tags:
	- Python
	- Jupyter
	- SourceCode
description:
---

在這邊文章前，必須要先知道什麼是 **Jupyter Notebook**，這方面網路上已經有太多的文章在介紹了，所以這邊就簡單介紹就好。

Jupyter Notebook 是一個介於 IDE 與編輯器之間的工具，可以讓使用者一行一行的寫程式並且直接運行觀察其結果，除了大家都熟知的 Python/R 等直譯式語言外，現在連 C++ 這種編譯式語言都可以執行了，非常的令人驚艷，想要看更多關於 C++ 支援的可以參考這篇[文章](https://blog.jupyter.org/interactive-workflows-for-c-with-jupyter-fe9b54227d92)

這篇文章的主軸在於如何透過程式化的方式將已知的 **Jupyter Notebook** 給轉換成一般常用的 **Python** 腳本。

<!--more-->


基本上去網路上搜尋如何將 **Jupyter Notebook** 轉換成 Python 腳本，你都會得到使用下列指令的答案
```shell=
ipython nbconvert --to script notebook.ipynb
```
該指令會將該 **Jupyter notebook**檔案 **notebook.ipynb** 轉換成 **notebook.py**
這樣就可以透過 **python** 順利執行了。

但是事情通常都沒有這樣單純，事實上 **Jupyter notebook** 背後走的是 **IPython** 而非 **Python**，所以你可以在 **Jupyter notebook** 內採用 **!** 前綴的方式去執行系統指令，譬如
![](https://i.imgur.com/gBo7yz2.png)
所以你轉換出來的 **notebook.py** 本身也需要透過 **ipython** 的程式去執行，這在某些情況會造成不方便。
所以這時候就有一個想法出現了，因為 **ipython nbconvert** 可以用來轉換 **ipynb** 到 **py**，如果我們能夠在轉換的過程中，將 **ipython** 的程式碼都遮蔽掉，是否就可以達到轉譯出來的腳本就是 **Python** 腳本了，而不需要使用 **IPython** 來執行。

此外，這整個方向要能夠正確的前提是原本的 **Jupyter Notebook** 內沒有使用 **IPython** 的結果來進行關鍵性的操作，這意味者即使去除所有 **IPython** 的程式碼也不能影響整體程式的運行結果。

為了研究這個方向，我決定採取下列步驟來研究是否有辦法完成
1. 研究 IPython 檔案內，**IPython** 程式碼跟 **Python** 是否有差異
2. 研究是否有好的處理邏輯能夠將 **IPython** 程式碼與 **Python** 分離
3. 研究 **ipython nbconvert** 是如何透過程式的方式去轉換這個檔案

因此下列就針對這三個步驟進行一些分析與過程的介紹

#### Study Jupyter Notebook file
隨便撰寫一個簡單的 **Jupyter Notebook**，其內容如下
![](https://i.imgur.com/DK422kE.png)

這時候透過 **Jupyter Notebook** 的下載工具將其轉換到不同格式，這邊我們嘗試了兩種格式，分別是 **ipynb**，另外一種則是 **py**。
所以上述的格式對於 **.py** 這種來看，則是
``` python

# coding: utf-8

# In[1]:


a=123


# In[2]:


print(a)


# In[3]:


get_ipython().system('ls')

```

假設若要針對 **python** 程式來轉換的話，我們可以大膽假設只要是**get_ipython()** 都屬於 **IPython** 的程式碼，所以我們可以寫一個程式碼讀取整個 **Python** 檔案，並且符合此特徵的程式碼給忽略，就可以將此 **Python** 變成純 **Python** 而不依賴於 **IPYthon** 了。
這種做法的話非常容易，就不提供相關的程式碼了，只要可以讀取檔案，過濾符合特徵的程式碼然後在存擋就可以完成。


那如果是另外一種 **.ipynb** 的格式的話，其格式長這樣
``` json
{
 "cells": [
  {
   "cell_type": "code",
   "execution_count": 1,
   "metadata": {},
   "outputs": [],
   "source": [
    "a=123"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 2,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "123\n"
     ]
    }
   ],
   "source": [
    "print(a)"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": 3,
   "metadata": {},
   "outputs": [
    {
     "name": "stdout",
     "output_type": "stream",
     "text": [
      "communities\t\t\t       Welcome to Haskell.ipynb\r\n",
      "datasets\t\t\t       Welcome to Python.ipynb\r\n",
      "featured\t\t\t       Welcome to Ruby.ipynb\r\n",
      "Untitled.ipynb\t\t\t       Welcome to Spark with Python.ipynb\r\n",
      "Welcome Julia - Intro to Gadfly.ipynb  Welcome to Spark with Scala.ipynb\r\n",
      "Welcome R  - demo.ipynb\t\t       work\r\n"
     ]
    }
   ],
   "source": [
    "!ls"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": []
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.6.3"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 2
}

```
這個格式是依照 **Json** 的格式來輸出，可以觀察到物件中若有 **source** 欄位時，其值就是該步驟的指令。
而根據前述的規範，若指令是依照 **!** 的前綴來撰寫都屬於 **IPython** 的，則我們可以判斷若該 **Source** 欄位的數值其開頭是 **!** 的，就屬於 **IPython** 的程式碼，可以將該 **Json** Obejct 給移除。
剛好在網路上看到也有類似的問題，提出的解法也很類似，參考這篇[gist](https://gist.github.com/hannorein/8423ad75547f37bba228)
```json
   "source": [
    "!ls"
   ]
   "source": [
    "print(a)"
   ]
```

看到這邊，不禁想到開頭所述的轉換指令，該指令可以將 **.ipynb** 轉換到 **.py** 的檔案
```shell=
ipython nbconvert --to script notebook.ipynb
```
這意味者 **ipython nbconvert** 本身可以讀取這種 json 物件，並且將其轉換成 **.py** 的格式。
這樣我們就可以在中間讀取 **Json** 物件時，將檔案內的關於 **IPython** 的物件給移除，剩下就繼續處理即可。

所以接下來我們就來研究到底 **ipython nbconvert** 這個程式到底怎麼運作的。
首先，找到其 [GitHub](https://github.com/jupyter/nbconvert/blob/master/nbconvert/nbconvertapp.py) 的檔案，再仔細看了一下後，我們找到了一個名為 **NbConvertApp** 的物件，然後觀察到該物件有一個 **Start** 的函式

```python
class NbConvertApp(JupyterApp):
    """Application used to convert from notebook file type (``*.ipynb``)"""
    
    version = __version__
    name = 'jupyter-nbconvert'
    aliases = nbconvert_aliases
    flags = nbconvert_flags
    ....
```

```python
    def start(self):
        """Run start after initialization process has completed"""
        super(NbConvertApp, self).start()
        self.convert_notebooks()
```

接下來我們仔細看一下 **convert_notebooks** 這個函式, 這個函式的內容頗長，大概看一下包含了
```python

    def convert_notebooks(self):
        """Convert the notebooks in the self.notebook traitlet """
        # check that the output base isn't specified if there is more than
        # one notebook to convert
        if self.output_base != '' and len(self.notebooks) > 1:
            self.log.error(
                """
                UsageError: --output flag or `NbConvertApp.output_base` config option
                cannot be used when converting multiple notebooks.
                """
            )
            self.exit(1)
        
        # initialize the exporter
        cls = get_exporter(self.export_format)
        self.exporter = cls(config=self.config)

        # no notebooks to convert!
        if len(self.notebooks) == 0 and not self.from_stdin:
            self.print_help()
            sys.exit(-1)

        # convert each notebook
        if not self.from_stdin:
            for notebook_filename in self.notebooks:
                self.convert_single_notebook(notebook_filename)
        else:
            input_buffer = unicode_stdin_stream()
            # default name when conversion from stdin
            self.convert_single_notebook("notebook.ipynb", input_buffer=input_buffer)
```
我們首先會觀察到，必須要先初始化一個 **self.exporter**，這個 **exporter** 意味者當前的轉換要轉換到什麼樣的格式，譬如 **script** 就是轉換到 **python script**。

接下來會根據你當前的轉換來源是哪邊，若你是透過 **stdin** 的方式傳輸內容近來，則會透過 ** input_buffer = unicode_stdin_stream()** 來處理，否則就會根據輸入的檔案名稱來選擇
```python
self.convert_single_notebook(notebook_filename)
```
所以在仔細檢查該函式，就會觀察到底下的事情分成四個部分去執行
```python
    def convert_single_notebook(self, notebook_filename, input_buffer=None):
        """Convert a single notebook.
        Performs the following steps:
            1. Initialize notebook resources
            2. Export the notebook to a particular format
            3. Write the exported notebook to file
            4. (Maybe) postprocess the written file
        Parameters
        ----------
        notebook_filename : str
        input_buffer :
            If input_buffer is not None, conversion is done and the buffer is
            used as source into a file basenamed by the notebook_filename
            argument.
        """
        if input_buffer is None:
            self.log.info("Converting notebook %s to %s", notebook_filename, self.export_format)
        else:
            self.log.info("Converting notebook into %s", self.export_format)
        
        resources = self.init_single_notebook_resources(notebook_filename)
        output, resources = self.export_single_notebook(notebook_filename, resources, input_buffer=input_buffer)
        write_results = self.write_single_notebook(output, resources)
        self.postprocess_single_notebook(write_results)
```
分別是程式碼內註解所描述的行為
1. Initialize notebook resources
2. Export the notebook to a particular format
3. Write the exported notebook to file
4. (Maybe) postprocess the written file

我們可以忽略第四個行為，專注於前面三個步驟，所以我們就要仔細研究上述三個行為對應的函式。
#### Initialize notebook resources
首先可以看到在 **init_single_notebook_resources** 的函式內，我們要想辦法產生一個 **map** 的物件，該物件按照說明，有三個 **key** 需要填寫，不過因為我們只是單純要進行轉移，所以其實 **config_dir** 這個 key 並不需要使用。
```python
    def init_single_notebook_resources(self, notebook_filename):
        """Step 1: Initialize resources
        This initializes the resources dictionary for a single notebook.
        Returns
        -------
        dict
            resources dictionary for a single notebook that MUST include the following keys:
                - config_dir: the location of the Jupyter config directory
                - unique_key: the notebook name
                - output_files_dir: a directory where output files (not
                  including the notebook itself) should be saved
        """
        basename = os.path.basename(notebook_filename)
        notebook_name = basename[:basename.rfind('.')]
        if self.output_base:
            # strip duplicate extension from output_base, to avoid Basename.ext.ext
            if getattr(self.exporter, 'file_extension', False):
                base, ext = os.path.splitext(self.output_base)
                if ext == self.exporter.file_extension:
                    self.output_base = base
            notebook_name = self.output_base

        self.log.debug("Notebook name is '%s'", notebook_name)

        # first initialize the resources we want to use
        resources = {}
        resources['config_dir'] = self.config_dir
        resources['unique_key'] = notebook_name

        output_files_dir = (self.output_files_dir
                            .format(notebook_name=notebook_name))

        resources['output_files_dir'] = output_files_dir

        return resources
``` 
#### Export the notebook to a particular format
接下來就是要把來源的檔案給放到 **Exporter** 去處理格式轉換的問題，但是這邊如果我們直接傳入的是一個本來的 **.ipynb** 的檔案的話，我們會沒有辦法能夠針對 **IPython** 部份去進行修改。

因此這邊我們必須要考慮 **input_buffer** 這個參數。我們要先自己讀取該檔案，將該檔案的內容解析完畢後，排除 **IPython** 相關的內容後當成 **input_buffer** 參數給丟進去處理。

```python
 def export_single_notebook(self, notebook_filename, resources, input_buffer=None):
        """Step 2: Export the notebook
        Exports the notebook to a particular format according to the specified
        exporter. This function returns the output and (possibly modified)
        resources from the exporter.
        Parameters
        ----------
        notebook_filename : str
            name of notebook file.
        resources : dict
        input_buffer :
            readable file-like object returning unicode.
            if not None, notebook_filename is ignored
        Returns
        -------
        output
        dict
            resources (possibly modified)
        """
        try:
            if input_buffer is not None:
                output, resources = self.exporter.from_file(input_buffer, resources=resources)
            else:
                output, resources = self.exporter.from_filename(notebook_filename, resources=resources)
        except ConversionException:
            self.log.error("Error while converting '%s'", notebook_filename, exc_info=True)
            self.exit(1)

        return output, resources
```
#### Write the exported notebook to file        
在上述已經將內容轉換完畢後，就可以透過最後的 **writer** 將該內容(放在 **output**)內給寫入到特定的地方。

```python
   def write_single_notebook(self, output, resources):
        """Step 3: Write the notebook to file
        This writes output from the exporter to file using the specified writer.
        It returns the results from the writer.
        Parameters
        ----------
        output :
        resources : dict
            resources for a single notebook including name, config directory
            and directory to save output
        Returns
        -------
        file
            results from the specified writer output of exporter
        """
        if 'unique_key' not in resources:
            raise KeyError("unique_key MUST be specified in the resources, but it is not")

        notebook_name = resources['unique_key']
        if self.use_output_suffix and not self.output_base:
            notebook_name += resources.get('output_suffix', '')

        write_results = self.writer.write(
            output, resources, notebook_name=notebook_name)
        return write_results
```        

所以看完上述的整理流程後，我們可以整理一下整體的 **psuedo code** 以及相關的安裝環境。
```shell=
pip install ipython
```

```python
data=read_file("xxxx.ipynb")
parse_json(data)
filter_ipython(data)

#get the exporter for script format
cls = get_exporter("script")

#initial the resources
init_single_notebook_resources(...)
#export the data to python script
export_single_notebook(...,data)
#writhe the output to file
write_single_notebook(...)
```

由於這部分不會太麻煩，所以這邊就列了一個大概的 **pseudo code**，經過實際測試也是真的可以運作的。

整篇文章到這邊也就要結束了，主要是練習直接透過程式碼的方式進行 **Jupyter Notebook** 的轉換，同時也能夠有中間資料過濾的能力。


