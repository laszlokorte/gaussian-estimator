<script>
  import Gauss from "./Gauss.svelte";
  import { Canvas, Layer, t } from "svelte-canvas";

  let w, h;
  let samples = []
  let view3d = 'no'

  let sampleLimit = 0

  $: vmin = Math.min(w,h)

  const formatter = new Intl.NumberFormat(navigator.locale, { maximumFractionDigits: 2, minimumFractionDigits: 2,signDisplay: 'always' })

  $: usedSamples = samples.slice(0, sampleLimit)

  $: mean = {
  	x: usedSamples.reduce((a, s) => s.x + a, 0) / (usedSamples.length||1),
  	y: usedSamples.reduce((a, s) => s.y + a, 0) / (usedSamples.length||1),
  }

  $: variance = {
  	x: usedSamples.reduce((a, s) => Math.pow(s.x - mean.x, 2) + a, 0) / ((usedSamples.length - 1) || 1),
  	y: usedSamples.reduce((a, s) => Math.pow(s.y - mean.y, 2) + a, 0) / ((usedSamples.length - 1) || 1),
  }

  $: covariance = usedSamples.reduce((a, s) => (s.x-mean.x)*(s.y-mean.y) + a, 0) / ((usedSamples.length-1)||1)
  $: correlation = covariance / (Math.sqrt(variance.x * variance.y) || 1)

  $: crossCovariance = [
  	variance.x, covariance,
  	covariance, variance.y
  ]

  function eigenValues([a,b,c,d]) {
  	return [
  		(a+d) / 2 + Math.sqrt((a+d)*(a+d)/4 - (a*d-b*c)),
  		(a+d) / 2 - Math.sqrt((a+d)*(a+d)/4 - (a*d-b*c))
  	]
  }

  function eigenVector(lambda, [a,b,c,d]) {
  	return [d-lambda, -c]
  }

  function eigenAngle(lambda, [a,b,c,d]) {
  	if(b==0 && a>=c) {
  		return 0
  	} else if(b==0 && a<c) {
  		return Math.PI
  	} else {
  		return Math.atan2(lambda-a,b)
  	}
  }

  $: eigenVals = eigenValues(crossCovariance)
  $: eigenVecs = eigenVals.map(lambda => eigenVector(lambda, crossCovariance))

  $: theta = eigenAngle(Math.max(...eigenVals), crossCovariance)

  $: render = ({ context, width, height }) => {
  	const vmin = Math.min(window.innerWidth, window.innerHeight)
  	context.beginPath()
  	context.lineWidth = 1;
  	context.strokeStyle = 'black'
  	const coordPadding = 20
  	const arrowWidth = 5
  	const arrowLength = 10
  	const coordLength = 50
    context.font = 'italic 10pt sans-serif';

  	context.moveTo(coordPadding,coordPadding)
  	context.lineTo(coordPadding,coordLength)
  	context.lineTo(coordPadding-arrowWidth,coordLength-arrowLength)
  	context.moveTo(coordPadding,coordLength)
  	context.lineTo(coordPadding+arrowWidth,coordLength-arrowLength)
  	context.moveTo(coordPadding,coordPadding)
  	context.lineTo(coordLength,coordPadding)
  	context.lineTo(coordLength-arrowLength,coordPadding-arrowWidth)
  	context.moveTo(coordLength,coordPadding)
  	context.lineTo(coordLength-arrowLength,coordPadding+arrowWidth)
  	context.stroke()

  	context.textAlign = 'center'
  	context.textBaseline = 'middle'
  	context.fillStyle = '#5c5'
  	context.fillText('X',coordLength + arrowLength,coordPadding)
  	context.fillStyle = '#c55'
  	context.fillText('Y',coordPadding,coordLength + arrowLength)

	context.fillStyle = `#88f`;
    context.beginPath();
    for (var i = usedSamples.length - 1; i >= 0; i--) {
    	let s = usedSamples[i]
    	context.moveTo(s.x*vmin, s.y*vmin)
    	context.arc(s.x*vmin, s.y*vmin, 3, 0, Math.PI * 2);
    }
    context.fill();

    context.beginPath();
	context.fillStyle = `#c88`;
    for (var i = usedSamples.length - 1; i >= 0; i--) {
    	let s = usedSamples[i]
    	context.moveTo(5, s.y*vmin)
    	context.arc(5, s.y*vmin, 3, 0, Math.PI * 2);
    }
    context.fill();

    context.beginPath();
	context.fillStyle = `#8c8`;
    for (var i = usedSamples.length - 1; i >= 0; i--) {
    	let s = usedSamples[i]
    	context.moveTo(s.x*vmin, 5)
    	context.arc(s.x*vmin, 5, 3, 0, Math.PI * 2);
    }
    context.fill();

	context.fillStyle = `#aaa2`;
	context.strokeStyle = `#0003`;
	context.lineWidth = 3;

	context.moveTo(vmin*mean.x, vmin*mean.y)

  context.beginPath();
	context.ellipse(
		mean.x*vmin, 
		mean.y*vmin, 
		vmin*Math.sqrt(Math.abs(Math.max(...eigenVals))), 
		vmin*Math.sqrt(Math.abs(Math.min(...eigenVals))), 
		theta, 0, Math.PI * 2);
    context.fill();
    context.stroke()

	context.moveTo(vmin*mean.x, vmin*mean.y)
    context.beginPath();
	context.fillStyle = `#aaa`;
	context.ellipse(mean.x*vmin, mean.y*vmin, 3, 3, 0, 0, Math.PI * 2);
    context.fill();
    context.moveTo(mean.x*vmin, mean.y*vmin)
	context.strokeStyle = `#aaa`;
	context.lineWidth = 2;
    context.beginPath();
    context.moveTo(mean.x*vmin, mean.y*vmin)
    context.lineTo(vmin*(mean.x+Math.sqrt(eigenVals[0])*(eigenVecs[0][0])/Math.sqrt(eigenVecs[0][0]*eigenVecs[0][0] + eigenVecs[0][1]*eigenVecs[0][1])), vmin*(mean.y+Math.sqrt(eigenVals[0])*(eigenVecs[0][1])/Math.sqrt(eigenVecs[0][0]*eigenVecs[0][0] + eigenVecs[0][1]*eigenVecs[0][1])))
    context.moveTo(vmin*mean.x, vmin*mean.y)

    context.lineTo(vmin*(mean.x+Math.sqrt(eigenVals[1])*(eigenVecs[1][0])/Math.sqrt(eigenVecs[1][0]*eigenVecs[1][0] + eigenVecs[1][1]*eigenVecs[1][1])), vmin*(mean.y+Math.sqrt(eigenVals[1])*(eigenVecs[1][1])/Math.sqrt(eigenVecs[1][0]*eigenVecs[1][0] + eigenVecs[1][1]*eigenVecs[1][1])))

    context.stroke()


    context.font = 'italic 12pt sans-serif';
    context.fillText("λ₁", 
    	vmin*(mean.x+ 0.5 * Math.sqrt(eigenVals[0])*(eigenVecs[0][0])/Math.sqrt(eigenVecs[0][0]*eigenVecs[0][0] + eigenVecs[0][1]*eigenVecs[0][1])
    	    	+ 18 * (eigenVecs[1][1])/Math.sqrt(eigenVecs[1][0]*eigenVecs[1][0] + eigenVecs[1][1]*eigenVecs[1][1])
    	    	), 
    	vmin*(mean.y+ 0.5 * Math.sqrt(eigenVals[0])*(eigenVecs[0][1])/Math.sqrt(eigenVecs[0][0]*eigenVecs[0][0] + eigenVecs[0][1]*eigenVecs[0][1])
    	    	+ 18 * (eigenVecs[1][0])/Math.sqrt(eigenVecs[1][0]*eigenVecs[1][0] + eigenVecs[1][1]*eigenVecs[1][1]))
    )

    context.fillText("λ₂", 
    	vmin*(mean.x+ 0.5 * Math.sqrt(eigenVals[1])*(eigenVecs[1][0])/Math.sqrt(eigenVecs[1][0]*eigenVecs[1][0] + eigenVecs[1][1]*eigenVecs[1][1])
    	    	+ 18 * (eigenVecs[0][1])/Math.sqrt(eigenVecs[0][0]*eigenVecs[0][0] + eigenVecs[0][1]*eigenVecs[0][1])), 
    	vmin*(mean.y+ 0.5 * Math.sqrt(eigenVals[1])*(eigenVecs[1][1])/Math.sqrt(eigenVecs[1][0]*eigenVecs[1][0] + eigenVecs[1][1]*eigenVecs[1][1])
    	    	+ 18 * (eigenVecs[0][0])/Math.sqrt(eigenVecs[0][0]*eigenVecs[0][0] + eigenVecs[0][1]*eigenVecs[0][1]))
    )


    if(usedSamples.length) {
		context.fillStyle = `#aaa`;
		context.strokeStyle = `#aaa`;
		context.lineWidth = 2;
	    context.beginPath();
	    context.moveTo(vmin*(mean.x - Math.sqrt(variance.x)), coordPadding);
	    context.lineTo(vmin*(mean.x + Math.sqrt(variance.x)), coordPadding);
	    context.moveTo(coordPadding,vmin*(mean.y - Math.sqrt(variance.y)));
	    context.lineTo(coordPadding,vmin*(mean.y + Math.sqrt(variance.y)));
	    context.stroke()    
	    context.moveTo(vmin*(mean.x - Math.sqrt(variance.x)), coordPadding);
		context.ellipse(vmin*mean.x, coordPadding, 3, 3, 0, 0, Math.PI * 2);
	    context.moveTo(coordPadding, vmin*(mean.y - Math.sqrt(variance.y)));
		context.ellipse(coordPadding, vmin*mean.y, 3, 3, 0, 0, Math.PI * 2);
	    context.fill();

	    context.fillStyle = `#8c8`
	    context.font = 'italic 12pt sans-serif';
	    context.fillText('μX', vmin*mean.x, 35)
	    context.fillText('√σX', mean.x - 20 - Math.sqrt(variance.x), 20)
	    context.fillStyle = `#c88`
	    context.fillText('μY', 35, vmin*mean.y)
	    context.fillText('√σY', 20, vmin*(mean.y - 20 - Math.sqrt(variance.y)))

    }

  };

  function addSample(evt) {
  	const vmin = Math.min(window.innerWidth, window.innerHeight)
  	samples = [...samples.slice(0, sampleLimit), {x: evt.pageX/vmin, y: evt.pageY/vmin}]
  	sampleLimit = samples.length
  }

  function clearSamples() {
  	samples = []
  	sampleLimit = samples.length

  }
</script>

<style>
	.container {
		display: grid;
		grid-template-columns: [all-start intro-start gl-start] minmax(min-content,1fr) minmax(min-content,1fr) minmax(min-content,2fr) [gl-end intro-end right-start] minmax(min-content,2fr) [right-end all-end];
		grid-template-rows: [all-start right-start gl-start intro-start] 1fr 1fr 1fr  [intro-end gl-end all-end right-end];
		gap:  1em;
		box-sizing: border-box;
		height: 100%;
		justify-items: stretch;
	}


	:global(body, html, #root) {
		height: 100%;
	}

	.intro {
		grid-area: intro;
		align-self: stretch;
		justify-self: stretch;
		pointer-events: none;
		font-style: italic;
		color: #888;
		display: flex;
		align-items: center;
		justify-content: center;
	}

	.scalebase {
		grid-area: intro;
		pointer-events: none;	
		align-self: stretch;
		justify-self: stretch;
	}

	.menu {
		grid-area: right;
		overflow: hidden;
		overflow-y:  auto;
		max-height: 100%;
		max-width: 100%;
		box-sizing: border-box;
	}

	.menu-inner {
		background-color: #fffa;
		pointer-events: all;
		padding: 1em 2em 1em 1em;
	}

	:global(canvas) {
		grid-area: all;
		width: 100%;
		height: 100%;
		max-width: 100%;
	}

	:global(.gl) {
		grid-area: gl;
		width: 100%;
		height: 100%;
	}

	:global(body) {
		margin: 0;
		padding:  0;
	}

	dl {
		display: grid;
		grid-template-columns: [full-start key-start] max-content [key-end value-start] auto [value-end full-end];
		gap: 0.5em 1em;
	}

	dt {
		grid-column: key;
	}

	dd {
		grid-column: value;
		margin: 0;
		justify-self: end;
		text-align: right;
	}

	.plainlist  {
		list-style: none;
		margin:0;
		padding: 0;
	}

  input {
    margin: 0;
  }

  .hidden {
  	display: none;
  }


	@media(max-width: 60em) {
		.container {
			grid-template-columns: [all-start intro-start gl-start right-start] 1fr [right-end all-end gl-end intro-end];
			grid-template-rows: [all-start gl-start intro-start] 60vmin [intro-end gl-end all-end right-start] auto  [right-end];
			height: unset;
			gap:  0;
			max-height: unset;
			align-items: stretch;
		}

		.menu {
			height: unset;
		}
	}
</style>

<div class="container">
	<Canvas style={`display: ${view3d != 'no' ? 'none':'block'}`} width={w} height={h} on:click={addSample}>
	  <Layer {render} />
	</Canvas>

  <Gauss addSample={addSample} class="gl" view={view3d} show={view3d != 'no'} mean={mean} variance={variance} correlation={correlation} samples={usedSamples.map(({x,y}) => [2*x,2*y])} />

	<div bind:clientWidth={w} bind:clientHeight={h} class="intro" class:hidden={samples.length > 0}>
		Click anywere to place some sample points.
	</div>

	<div bind:clientWidth={w} bind:clientHeight={h} class="scalebase">
	</div>

	<div class="menu">
		<div class="menu-inner">
			<h1>Gaussian Estimator</h1>
			<p>
				Click anywhere to place points/samples.
			</p>
			<p>
				Parameters of a Gaussian distribution will be estimated. 
			</p>
			<dl>
				<dt>Number of samples</dt>
				<dd>{sampleLimit}/{samples.length}</dd>
			</dl>
	    <dl on:click={(evt) => evt.stopPropagation()}>
	      <dt><label for="history">History</label></dt>
	      <dd><input disabled={samples.length < 1} id="history" type="range" min="0" max={samples.length} bind:value={sampleLimit}></dd>
	      <dt>3D</dt>
	      <dd style="display: flex; gap: 1em">
	        <label><input type="radio" bind:group={view3d} name="3d" value="no"> None</label>
	        <label><input type="radio" bind:group={view3d} name="3d" value="pdf"> PDF</label>
	        <label><input type="radio" bind:group={view3d} name="3d" value="cdf"> CDF</label>
	      </dd>
	    </dl>
			<h2>Estimations</h2>
			<dl>
				<dt>Mean(X)</dt>
				<dd>&mu;<sub>X</sub> = {formatter.format(vmin*mean.x)}</dd>
				<dt>Mean(Y)</dt>
				<dd>&mu;<sub>Y</sub> = {formatter.format(vmin*mean.y)}</dd>
				<dt>Variance(X)</dt>
				<dd>&sigma;<sub>X</sub> = {formatter.format(vmin*variance.x)}</dd>
				<dt>Variance(Y)</dt>
				<dd>&sigma;<sub>Y</sub> = {formatter.format(vmin*variance.y)}</dd>
				<dt>Standard Deviation(X)</dt>
				<dd>&Sqrt;&sigma;<sub>X</sub> = {formatter.format(vmin*Math.sqrt(variance.x))}</dd>
				<dt>Standard Deviation(Y)</dt>
				<dd>&Sqrt;&sigma;<sub>Y</sub> = {formatter.format(vmin*Math.sqrt(variance.y))}</dd>
				<dt>Covariance(X,Y)</dt>
				<dd>K<sub>XY</sub> = K<sub>YX</sub> ={formatter.format(vmin*covariance)}</dd>
				<dt>Correlation(X,Y)</dt>
				<dd>&rho;<sub>XY</sub> = {formatter.format(vmin*correlation)}</dd>
				<dt>CovarianceMatrix(X,Y)</dt>
				<dd><span style="white-space: nowrap;">&Sigma;<sub>XY</sub> = {formatter.format(vmin*crossCovariance[0])}; {formatter.format(vmin*crossCovariance[1])}</span><br><span style="white-space: nowrap;">{formatter.format(vmin*crossCovariance[2])}; {formatter.format(vmin*crossCovariance[3])}</span></dd>
				<dt>&Sigma;<sub>XY</sub> eigenvalues</dt>
				<dd>&lambda;<sub>1</sub> = {formatter.format(vmin*eigenVals[0])}</dd>
				<dd>&lambda;<sub>2</sub> = {formatter.format(vmin*eigenVals[1])}</dd>
				<dt>&Sigma;<sub>XY</sub> eigenvectors</dt>
				<dd style="white-space: nowrap;">v<sub>1</sub> = [{eigenVecs[0].map(x=>x*vmin).map(formatter.format).join(';')}]</dd>
				<dd style="white-space: nowrap;">v<sub>2</sub> = [{eigenVecs[1].map(x=>x*vmin).map(formatter.format).join(';')}]</dd>
			</dl>
			<h2>Calculation</h2>
			<ul class="plainlist">
				<li><code>&mu;<sub>X</sub> = sum(X)/length(X)</code></li>

				<li><code>&mu;<sub>Y</sub> = sum(Y)/length(Y)</code></li>

				<li><code>&sigma;<sub>X</sub> = sum(X - &mu;<sub>X</sub>)/(length(X) - 1)</code></li>

				<li><code>&sigma;<sub>Y</sub> = sum(Y - &mu;<sub>Y</sub>)/(length(Y) - 1)</code></li>

				<li><code>K<sub>XY</sub> = <code>K<sub>YX</sub> = sum((X-&mu;<sub>X</sub>)&middot;(Y-&mu;<sub>Y</sub>))/(length(X)-1)</code></li>

				<li><code>&rho;<sub>XY</sub> = K<sub>XY</sub> / sqrt(&sigma;<sub>X</sub>&middot;&sigma;<sub>Y</sub>)</code></li>

				<li><code>&Sigma;<sub>XY</sub> = [&sigma;<sub>X</sub>, K<sub>XY</sub>, &sigma;<sub>Y</sub>, K<sub>YX</sub>]</code></li>

				<li><code>&lambda;<sub>1,2</sub> = (&sigma;<sub>X</sub>+&sigma;<sub>Y</sub>)/2 &plusmn; sqrt((&sigma;<sub>X</sub>+&sigma;<sub>Y</sub>)<sup>2</sup>/4 - (&sigma;<sub>X</sub>&middot;&sigma;<sub>Y</sub>-K<sub>XY</sub>&middot;K<sub>YX</sub>))</code></li>

			</ul>
		</div>
	</div>
</div>