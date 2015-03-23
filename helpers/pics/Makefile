default: latex.png numa.png membench_xeon_phi.png netio_phi.png pi.png

latex.png: latex.tex
	pdflatex -shell-escape latex.tex

membench_xeon_phi.png: membench_xeon_phi.tex
	pdflatex -shell-escape membench_xeon_phi.tex

netio_phi.png: netio_phi.tex
	pdflatex -shell-escape netio_phi.tex

netio_mic2mic.png: netio_mic2mic.tex
	pdflatex -shell-escape netio_mic2mic.tex

pi.png: pi.tex
	pdflatex -shell-escape pi.tex

numa.png: numa.tex
	pdflatex -shell-escape numa.tex

clean:
	rm -rf *.png *.log *.pdf *.aux
