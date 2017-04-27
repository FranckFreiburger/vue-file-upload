<template>
	<div class="root" :class="dropState ? 'drop_'+dropState : ''">
		<form :target="ifrList[0]" :action="url" method="post" enctype="multipart/form-data">
			<input :id="'inp'+_uid" name="file" type="file" accept="image/*;capture=camera" @change="onchange" :multiple="multiple">
		</form>
	 	<iframe v-for="name in ifrList" :key="name" :name="name" src="about:blank" @load="onload($event.currentTarget, name)"></iframe>
		<label :for="!uploading || multiple ? 'inp'+_uid : ''" @dragenter.prevent.stop="enter" @dragleave.prevent.stop="leave" @dragover.prevent.stop="over" @drop.prevent.stop="drop"></label>
		<div v-show="uploading" class="progressBar">
			<div class="progress" :style="{width: progress*100+'%'}"></div>
			<a href="#" class="cancel" @click.prevent="cancel"></a>
		</div>
	</div>
</template>
<style scoped>
	.root {
		position: relative;
		width: 10em;
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

function hasFileAPI() {
	
	return false;
	
	return window.FileReader && window.FormData;
}

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
	components: {
		ifr: {
			methods: {
				onload: function(iframeElt, name) {

					if ( !isIFrameInitState(iframeElt) )
						this.$emit('ifrLoaded', name);
				}
			}
		}
	},
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
			hasFileAPI: hasFileAPI(), //  false,// 
			dropState: '',
			uploading: false,
			progress: 0,
			ifrList: [],
			ifrId: 1,
		}
	},
	
	methods: {
		uploaded: function(status) {
			
			this.uploading = false;
		},
		uploadFile: function(file) {
			
			var xhr = new XMLHttpRequest();
			
			var cancel = function() {

				xhr.abort();
			};
			
			xhr.upload.onprogress = function(e) {
				
				this.progress = e.lengthComputable ? e.loaded / e.total : 0.5;
			}.bind(this);

			xhr.onreadystatechange = function() {
				
				if ( xhr.readyState !== 4 )
					return;
				xhr.onreadystatechange = null;
				xhr.upload.onprogress = null;
				this.$off('cancel', cancel);
				
				this.uploaded(xhr.status); // xhr.responseText
			}.bind(this);
			xhr.open('POST', this.url, true);
			
			var fd = new FormData();
			fd.append('file', file);
			xhr.send(fd);
			
			this.$once('cancel', cancel);
		},
		enter: function(ev) {
			
			if ( this.uploading && !this.multiple )
				return;

			if ( this.hasFileAPI ) {
				
				if ( ('items' in ev.dataTransfer) && ev.dataTransfer.items.length > 1 && !this.multiple )
					this.dropState = 'denied';
				else
					this.dropState = 'allowed';
			}
		},
		leave: function(ev) {

			if ( this.uploading && !this.multiple )
				return;

			if ( this.hasFileAPI )
				this.dropState = '';
		},
		over: function(ev) {

			if ( this.uploading && !this.multiple )
				return;
			
			if ( this.hasFileAPI )
				ev.dataTransfer.dropEffect = (this.dropState === 'denied' ? 'none' : '');
		},
		drop: function(ev) {

			if ( this.uploading && !this.multiple )
				return;

			if ( this.hasFileAPI ) {
				
				this.clearDropState();
				
				if ( ev.dataTransfer.files.length === 0 || ev.dataTransfer.files.length > 1 && !this.multiple ) {
					
					this.dropState = 'denied';
				} else {
					
					this.uploadFile(ev.dataTransfer.files[0]);
					this.uploading = true;
				}
			} else {
			
				this.$nextTick(function() {

					var inputElt = document.getElementById(ev.target.htmlFor);
					inputElt.click();
				});
			}
		},
		onchange: function(ev) {
			
			this.uploading = true;
			
			if ( this.hasFileAPI ) {
				
				this.uploadFile(ev.currentTarget.files[0]);
			} else {
				
				this.progress = 0.5;
				
				this.ifrId++;
				this.ifrList.unshift(this._uid+'ifr'+this.ifrId);

				this.$nextTick(function() {
					
					var formElt = ev.target.form;
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
			this.ifrList.splice(this.ifrList.indexOf(name), 1);
		},
		
		cancel: function() {
			
			this.$emit('cancel');
			
			this.ifrList.splice(0, this.ifrList.length);
			this.uploading = false;
		},
		clearDropState: function() {
			
			window.setTimeout(function() {
				
				this.dropState = '';
			}.bind(this), 500);
		}
	}
}

</script>