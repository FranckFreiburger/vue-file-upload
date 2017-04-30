<template>
	<div class="root" :class="[dropState, uploadState]">
		<form :target="uploads.length && uploads[0].ifr" :action="url" method="post" enctype="multipart/form-data">
			<input :id="'inp'+_uid" name="file" type="file" :accept="image ? 'image/*' : undefined" :capture="capture" @change="onchange" :multiple="multiple">
		</form>
	 	<iframe v-for="item in uploads" v-if="item.ifr" :key="item.ifr" :name="item.ifr" src="about:blank" @load="onload($event.target, item.ifr)"></iframe>
	 	<div v-if="$slots.default" class="slot"><slot></slot></div>
	 	<div class="notice"></div>
		<label :for="!uploads.length || multiple ? 'inp'+_uid : ''" @dragenter.prevent.stop="enter" @dragleave.prevent.stop="leave" @dragover.prevent.stop="over" @drop.prevent.stop="drop" :title="uploadInfo"></label>
		<div v-show="uploads.length" class="progressBar">
			<div class="progress" :style="progressStyle"></div>
			<a href="#" class="cancel" @click.prevent="free"></a>
		</div>
	</div>
</template>
<style scoped>

	@keyframes notice {
		from {
			opacity: 1;
		}
		to {
			opacity: 0;
		}
	}
	.root {
		position: relative;
		display: inline-block;
		min-width: 5em;
		min-height: 5em;
		background-color: #eee;
	}
	
	form,
	iframe {
		display: none;
	}
	
	.notice,
	.slot {
		position: absolute;
		top: 50%;
		left: 50%;
		transform: translate(-50%, -50%);
	}
	
	label,
	.root:before {
		position: absolute;
		top: 0;
		right: 0;
		bottom: 0;
		left: 0;
	}
	
	label {
		background-color: #000;
		opacity: 0;
		cursor: pointer;
	}
	
	.root:before {
		content: '';
		margin: 0.5em;
		border: 0.25em dotted silver;
	}
	
	.dropAllowed.root:before {
		border-color: green;
	}

	.dropDenied.root:before {
		border-color: red;
	}

	.notice {
		font-family: arial;
		font-weight: bold;
		font-size: 250%;
	}

	.uploadSuccess .notice,
	.uploadFailure .notice {
		animation-duration: 1s;
		animation-name: notice;
	}
	
	.uploadSuccess .notice:before {
		color: green;
		content: '\2713';
	}

	.uploadFailure .notice:before {
		color: red;
		content: '\2717';
	}
	
	
	
	.progressBar {
		position: absolute;
		left: 1em;
		bottom: 1em;
		right: 1em;
		height: 1em;
		background-color: silver;
	}
	
	.progress {
		height: 100%;
		background-color: grey;
	}
	
	.cancel {
		position: absolute;
		top: 50%;
		margin-top: -0.6em;
		height: 100%;
		right: 0;

		font-family: arial;
		font-weight: bold;
		color: red;
		text-decoration: none;
		cursor: pointer;
	}
	
	.cancel:before {
		content: '\00A0x\00A0';
	}
	
</style>
<script>
"use strict";

var hasFileAPI = window.FileReader && window.FormData;

//hasFileAPI = false

function isIFrameWindow(iframeElt) {
	
	try {
		return iframeElt.contentWindow;
	} catch(ex) {}
	return null;
}

function hasDataTransferFileSupport(dataTransfer) {
	
	if ( !('types' in dataTransfer) )
		return false;
	for ( var i = 0; i < dataTransfer.types.length; ++i )
		if ( dataTransfer.types[i] === 'Files' )
			return true;
	return false;
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
		},
		image: {
			type: Boolean,
			default: false,
		},
		capture: {
			type: Boolean,
			default: false,
		},
		check: {
			type: Function,
			default: function() { return true },
		},
		done: {
			type: Function,
			default: function(status, responseText, feedback) { status !== undefined && feedback(status >= 200 && status < 400) },
		}
	},
	data: function() {
		return {
			dropState: '',
			uploadState: '',
			uploads: [],
			total: 0,
			loaded: 0,
			tick: 0,
		}
	},
	
	computed: {
		uploadInfo: function() {
			
			var filenameList = [];
			for ( var i = 0; i < this.uploads.length; ++i )
				Array.prototype.push.apply(filenameList, this.uploads[i].filenameList);
			return filenameList.map(function(item) { return '\u2022 ' + item }).join('\n');
		},
		progressStyle: function() {
			
			if ( this.uploads.length === 0 )
				return undefined;
			
			var loadedRatio = this.loaded / this.total;
			if ( isNaN(loadedRatio) )
				return { width: '50%', marginLeft: (50 - Math.abs(this.tickNext() % 50*2 - 50))+'%' };
			else
				return { width: loadedRatio*100+'%' };
		},
	},
	
	watch: {
		'uploads.length': function(length) {
			
			if ( length !== 0 )
				return;
			this.loaded = 0;
			this.total = 0;
		}
	},
	
	methods: {
		tickNext: function() {

			window.setTimeout(function() {
			
				this.tick++;
			}.bind(this), 250);
			return this.tick;
		},
		
		uploaded: function(status, responseText) {
			
			var feedback = function(success) {
			
				this.uploadState = success ? 'uploadSuccess' : 'uploadFailure';
				window.setTimeout(function() {
				
					this.uploadState = '';
				}.bind(this), 1000);
			}.bind(this);
			
			this.done(status, responseText, feedback);
		},
		uploadFile: function(files) {
			
			var xhr = new XMLHttpRequest();

			var info = {
				free: function() {
					
					xhr.onreadystatechange = null;
					xhr.upload.onprogress = null;
					if ( xhr.readyState !== 4 )
						xhr.abort();
					this.uploads.splice(this.uploads.indexOf(info), 1);
				}.bind(this),
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
				info.free();
				this.uploaded(xhr.status, xhr.responseText);
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
			
			if ( this.uploads.length && !this.multiple ) {
				
				this.dropState = 'dropDenied';
				return;
			}

			if ( hasFileAPI && hasDataTransferFileSupport(ev.dataTransfer) ) {
				
				if ( ('items' in ev.dataTransfer) && ev.dataTransfer.items.length > 1 && !this.multiple )
					this.dropState = 'dropDenied';
				else
					this.dropState = 'dropAllowed';
			}
		},
		leave: function(ev) {

			this.dropState = '';
		},
		over: function(ev) {

			if ( this.uploads.length && !this.multiple ) {
				
				this.dropState = 'dropDenied';
				return;
			}
			
			if ( hasFileAPI && hasDataTransferFileSupport(ev.dataTransfer) )
				ev.dataTransfer.dropEffect = (this.dropState === 'dropDenied' ? 'none' : '');
		},
		
		drop: function(ev) {
			
			window.setTimeout(function() {
			
				this.dropState = '';
			}.bind(this), 500);

			if ( this.uploads.length && !this.multiple ) {
				
				this.dropState = 'dropDenied';
				return;
			}

			if ( hasFileAPI && hasDataTransferFileSupport(ev.dataTransfer) ) {
				
				var files = Array.prototype.slice.call(ev.dataTransfer.files);
				var acceptFiles = !files.some(function(file) {
					
					return !this.check(file.name);
				}.bind(this));
				
				if ( !acceptFiles ) {
					
					this.dropState = 'dropDenied';
					return;
				}
				
				if ( ev.dataTransfer.files.length === 0 || ev.dataTransfer.files.length > 1 && !this.multiple ) {
				
					this.dropState = 'dropDenied';
				} else {
					
					this.uploadFile(ev.dataTransfer.files);
				}
			} else {
			
				this.$nextTick(function() {

					if ( ev.target.htmlFor )
						document.getElementById(ev.target.htmlFor).click();
				});
			}
		},
		onchange: function(ev) {
			
			var formElt = ev.target.form;
			if ( hasFileAPI && 'files' in ev.target ) {
				
				this.uploadFile(ev.target.files);
				formElt.reset();
			} else {
				
				this.total += NaN;
				var name = this._uid+'ifr'+Date.now();

				var filename = ev.target.value.match(/[^\/\\]*$/)[0];
				if ( !this.check(filename) ) {
					
					formElt.reset();
					this.dropState = 'dropDenied';
					
					window.setTimeout(function() {
					
						this.dropState = '';
					}.bind(this), 500);
					
					return;
				}
				
				this.uploads.unshift({
					ifr: name,
					free: function() {
			
						for ( var i = 0; i < this.uploads.length; ++i )
							if ( this.uploads[i].ifr === name )
								this.uploads.splice(i, 1);
					}.bind(this),
					filenameList: [filename],
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
			
			var iframeWin = isIFrameWindow(iframeElt);
			if ( iframeWin !== null && iframeWin.document.location.href === 'about:blank' )
				return;
			
			for ( var i = 0; i < this.uploads.length; ++i )
				if ( this.uploads[i].ifr === name )
					this.uploads[i].free();

			this.uploaded(undefined, iframeWin !== null ? iframeWin.document.documentElement.innerText : '');
		},
		
		free: function() {
			
			for ( var i = this.uploads.length-1; i >= 0 ; --i )
				this.uploads[i].free();
		}
	}
}

</script>