---
layout: post
title: "How to make a proper double scrollbar frame in Tkinter"
date: 2015-07-21
---

A few times, while I was still working on [the Cryptopals challenges](http://cryptopals.com/) I needed to built a little GUI in order to help me analyze ciphertexts encrypted with the same AES-CTR key. 

<!--more-->

For that matter, I always use `Python` with `Tkinter` since it's much faster to prototype an interface with it than let's say `C\GTK` or `C++\MFC`. I needed to create a "grid" (like an spreadsheet) with the char of every ciphertexts laid out on top of each-other, so I could tweak one and see the result on the others char located at the same index.

It turns out, there is no "easy" way to implement a double scrollbar frame in Tkinter, so I had to come up with my own (Python 3.4) : 


{% highlight python tabsize=2 %}


import tkinter as tk
from tkinter import ttk


class DoubleScrollbarFrame(ttk.Frame):

	def __init__(self, master, **kwargs):
		'''
			Initialisation. The DoubleScrollbarFrame consist of :
			  - an horizontal scrollbar
			  - a  vertical   scrollbar
			  - a canvas in which the user can place sub-elements
		'''

		ttk.Frame.__init__(self,  master, **kwargs)

		# Canvas creation with double scrollbar
		self.hscrollbar = ttk.Scrollbar(self, orient = tk.HORIZONTAL)
		self.vscrollbar = ttk.Scrollbar(self, orient = tk.VERTICAL)
		self.sizegrip = ttk.Sizegrip(self)
		self.canvas = tk.Canvas(self, bd=0, highlightthickness=0, 
                                  yscrollcommand = self.vscrollbar.set,
                                  xscrollcommand = self.hscrollbar.set)
		self.vscrollbar.config(command = self.canvas.yview)
		self.hscrollbar.config(command = self.canvas.xview)

	def pack(self, **kwargs):
		'''
			Pack the scrollbar and canvas correctly in order to recreate the same
      look as MFC's windows. 
		'''

		self.hscrollbar.pack(side=tk.BOTTOM, fill=tk.X, expand=tk.FALSE)
		self.vscrollbar.pack(side=tk.RIGHT, fill=tk.Y,  expand=tk.FALSE)
        self.sizegrip.pack(in_ = self.hscrollbar, side = tk.BOTTOM, anchor = "se")
		self.canvas.pack(side=tk.LEFT, padx=5, pady=5,
                                             fill=tk.BOTH, expand=tk.TRUE)

		ttk.Frame.pack(self, **kwargs)
		


	def get_frame(self):
		'''
			Return the "frame" useful to place inner controls.
		'''
		return self.canvas


if __name__ == '__main__':

	# Top-level frame
	root = tk.Tk()
	root.title( "Double scrollbar with tkinter" )
	root.minsize(width = 600, height = 600) 	
	frame = DoubleScrollbarFrame(root, relief="sunken")

	# Add controls here
	subframe = ttk.Frame( frame.get_frame() ) 
	txt = ttk.Label(subframe, text="Add things here !")

	#Packing everything
	txt.pack(anchor = 'center', fill = tk.Y, expand = tk.Y)
	subframe.pack(padx  = 15, pady   = 15, fill = tk.BOTH, expand = tk.TRUE)
	frame.pack( padx   = 5, pady   = 5, expand = True, fill = tk.BOTH)


	# launch the GUI
	root.mainloop() 	
{% endhighlight python %}

![Exemple of a double scrollbar](/assets/DoubleScrollbarFrameExemple.PNG)

