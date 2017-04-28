<template>
	<div class="root" :class="dropState ? 'drop_'+dropState : ''">
		<form :target="uploads[0] && uploads[0].ifr" :action="url" method="post" enctype="multipart/form-data">
			<input :id="'inp'+_uid" name="file" type="file" accept="image/*" capture @change="onchange" :multiple="multiple">
		</form>
	 	<iframe v-for="item in uploads" v-if="item.ifr" :key="item.ifr" :name="item.ifr" src="about:blank" @load="onload($event.target, item.ifr)"></iframe>
		<label :for="!uploads.length || multiple ? 'inp'+_uid : ''" @dragenter.prevent.stop="enter" @dragleave.prevent.stop="leave" @dragover.prevent.stop="over" @drop.prevent.stop="drop" :title="uploadInfo"></label>
		<div v-show="uploads.length" class="progressBar">
			<div class="progress" :style="progressStyle"></div>
			<a href="#" class="cancel" @click.prevent="free"></a>
		</div>
	</div>
</template>
<style scoped>
	.root {
		position: relative;
		width: 5em;
		height: 5em;
	}
	
	form,
	iframe {
		display: none;
	}

	label {
		position: absolute;
		top: 0;
		right: 0;
		bottom: 0;
		left: 0;
		background-color: #eee;
		cursor: pointer;
	}
	
	label:before {
		content: '';
		position: absolute;
		top: 0;
		right: 0;
		bottom: 0;
		left: 0;
		
		margin: 0.5em;
		border: 0.25em dotted silver;
	}
	
	.drop_allowed label:before {
		border-color: green;
	}

	.drop_denied label:before {
		border-color: red;
	}
	
	label:after {
		position: absolute;
		top: 50%;
		left: 50%;
		width: 1em;
		height: 1em;
		margin: -0.5em 0 0 -0.5em;
		content: '\2713';
		font-weight: bold;
		font-size: 200%;
		color: green;
	}
	
	.progressBar {
		position: absolute;
		height: 0.75em;
		top: 60%;
		left: 1em;
		right: 1em;
		background-color: silver;
	}
	.progress {
		height: 100%;
		background-color: grey;
	}
	
	.cancel {
		position: absolute;
		right: 0;
		top: -0.25em;
		
		font-family: arial;
		font-weight: bold;
		color: red;
		text-decoration: none;
		cursor: pointer;
	}
	
	.cancel:before {
		content: '\02A2F'; /* &cross; */
	}
	
</style>
<script>
"use strict";

var hasFileAPI = window.FileReader && window.FormData;

//hasFileAPI = false

function isIFrameInitState(iframeElt) {
	
	try {
		return iframeElt.contentDocument.location.href === 'about:blank';
	} catch(ex) {}
	return false;
}

function hasDataTransferFileSupport(dataTransfer) {
	
	return dataTransfer.types !== undefined && Array.prototype.slice.call(dataTransfer.types).indexOf('Files') !== -1;
}

module.exports = {

	props: {
		url: {
			type: String,
			required: true,
		},
		multiple: {
			type: Boolean,
			default: false,
		}
	},
	data: function() {
		return {
			dropState: '',
			uploads: [],
			total: 0,
			loaded: 0,
			count: 0,
		}
	},
	
	computed: {
		uploadInfo: function() {
			
			var filenameList = [];
			for ( var i = 0; i < this.uploads.length; ++i )
				Array.prototype.push.apply(filenameList, this.uploads[i].filenameList);
			return filenameList.join('\n');
		},
		progressStyle: function() {
			
			var loadedRatio = this.loaded/this.total;
			if ( isNaN(loadedRatio) ) {
				
				return { width: '50%', marginLeft: (50 - Math.abs(this.countNext() % 50*2 - 50)  ) +'%' };
			} else {
				
				return { width: loadedRatio*100+'%' };
			}
		},
	},
	
	watch: {
		'uploads.length': function(length) {
			
			if ( length > 0 )
				return;
			this.loaded = 0;
			this.total = 0;
		}
	},
	
	methods: {
		countNext: function() {

			setTimeout(function() {
			
				this.count++;
			}.bind(this), 250);
			return this.count;
		},
		uploaded: function(status) {
			
		},
		uploadFile: function(files) {
			
			var xhr = new XMLHttpRequest();

			var info = {
				free: xhr.abort.bind(xhr), // will trigger onreadystatechange
				filenameList: [],
			}
			this.uploads.push(info);
			var prevLoadedBytes = 0;
			
			xhr.upload.onprogress = function(ev) {
				
				if ( !ev.lengthComputable )
					return;
				this.loaded += ev.loaded - prevLoadedBytes;
				prevLoadedBytes = ev.loaded;
			}.bind(this);

			xhr.onreadystatechange = function() {
				
				if ( xhr.readyState !== 4 )
					return;
				xhr.onreadystatechange = null;
				xhr.upload.onprogress = null;
				
				this.uploads.splice(this.uploads.indexOf(info), 1);
				
				this.uploaded(xhr.status); // xhr.responseText
			}.bind(this);
			xhr.open('POST', this.url, true);
			
			var fd = new FormData();
			
			for ( var i = 0; i < files.length; ++i ) {
				
				info.filenameList.push(files[i].name);
				this.total += files[i].size;
				fd.append('file', files[i]);
			}
			xhr.send(fd);
		},
		enter: function(ev) {
			
			if ( this.uploads.length && !this.multiple )
				return;

			if ( hasFileAPI ) {
				
				if ( ('items' in ev.dataTransfer) && ev.dataTransfer.items.length > 1 && !this.multiple )
					this.dropState = 'denied';
				else
					this.dropState = 'allowed';
			}
		},
		leave: function(ev) {

			this.dropState = '';
		},
		over: function(ev) {

			if ( this.uploads.length && !this.multiple )
				return;
			
			if ( hasFileAPI )
				ev.dataTransfer.dropEffect = (this.dropState === 'denied' ? 'none' : '');
		},
		drop: function(ev) {

			if ( this.uploads.length && !this.multiple )
				return;

			if ( hasFileAPI ) {
				
				this.clearDropState();
				
				if ( ev.dataTransfer.files.length === 0 || ev.dataTransfer.files.length > 1 && !this.multiple )
					this.dropState = 'denied';
				else
					this.uploadFile(ev.dataTransfer.files);
			} else {
			
				this.$nextTick(function() {

					if ( ev.target.htmlFor )
						document.getElementById(ev.target.htmlFor).click();
				});
			}
		},
		onchange: function(ev) {
			
			var formElt = ev.target.form;
			if ( hasFileAPI ) {
				
				this.uploadFile(ev.target.files);
				formElt.reset();
			} else {
				
				this.total = NaN;
				var name = this._uid+'ifr'+Date.now();
				
				this.uploads.unshift({
					ifr: name,
					free: function() {
			
						for ( var i = 0; i < this.uploads.length; ++i )
							if ( this.uploads[i].ifr === name )
								return this.uploads.splice(i, 1);
					}.bind(this),
					filenameList: [ev.target.value.match(/[^\/\\]*$/)[0]],
				});

				this.$nextTick(function() {
					
					formElt.submit();
					this.$nextTick(function() {
					
						formElt.reset();
					});
				});
			}
		},
		
		onload: function(iframeElt, name) {

			if ( isIFrameInitState(iframeElt) )
				return;
			
			for ( var i = 0; i < this.uploads.length; ++i )
				if ( this.uploads[i].ifr === name )
					return this.uploads[i].free();
		},
		
		free: function() {
			
			for ( var i = this.uploads.length-1; i >= 0 ; --i )
				this.uploads[i].free();
		},
		clearDropState: function() {
			
			window.setTimeout(function() {
				
				this.dropState = '';
			}.bind(this), 500);
		}
	}
}

</script>