<script>
	export default {
		onLaunch: function() {
			console.log('App Launch')
		},
		onShow: function() {
			console.log('App Show')
		},
		onHide: function() {
			console.log('App Hide')
		}
	}
</script>

<style>
	page {
		color: #19355f;
	}

	button {
		margin: 0;
		padding-left: 0;
		padding-right: 0;
	}

	button::after {
		border: none;
	}
</style>
