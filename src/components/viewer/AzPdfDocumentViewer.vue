<template>
    <div class="az-pdf-container" :class="customContainerClass" :style="{ height: height }">
        <Toolbar
            :disableButtons="!src"
            :downloadButton="downloadButton"
            :pagination="pagination"
            :printButton="printButton"
            :rotateButton="rotateButton"
            :scaleType="scale.type"
            @changeScaleType="changeScaleType"
            @zoomIn="zoomIn"
            @zoomOut="zoomOut"
            @rotate="rotate"
            @download="download"
            @print="print"
            v-show="!loadingPlaceHolder"
        />
        <div
            id="az-pdf-viewer"
            class="Viewer"
            :style="{ height: `calc(${height} - 62px)` }"
            v-show="!loadingPlaceHolder"
        >
            <div class="pdfViewer"></div>
        </div>
        <LoadingPlaceHolder :loading="loadingPlaceHolder" />
        <LoadingPrint :isPrinting="isPrinting" :printProgress="printProgress" @cancel="cancelPrint" />
    </div>
</template>

<script>
import Toolbar from './Toolbar'
import LoadingPlaceHolder from './LoadingPlaceHolder'
import LoadingPrint from './LoadingPrint'
import PrintService from './PrintService'
import PDFJSLib from 'pdfjs-dist/build/pdf'
import { PDFJS as PDFJSViewer } from 'pdfjs-dist/web/pdf_viewer.js'
export default {
    components: {
        Toolbar,
        LoadingPlaceHolder,
        LoadingPrint
    },
    mounted() {
        this.start()
        window.addEventListener('resize', this.updateScaleType)
    },
    destroyed() {
        window.removeEventListener('resize', this.updateScaleType)
    },
    watch: {
        src() {
            this.start()
        }
    },
    methods: {
        start() {
            this.startLoadingPlaceHolderIfNecessary()
            this.getPdfContainer()
            this.createEventBus()
            this.createPdfLinkService()
            this.createPdfViewer()
            this.renderDocument()
        },
        getPdfContainer() {
            this.pdf.container = document.querySelector('#az-pdf-viewer')
        },
        createEventBus() {
            const eventBus = new PDFJSViewer.EventBus()
            eventBus.on('pagesinit', this.pagesInitEventHandler)
            eventBus.on('scalechange', this.scaleChangeEventHandler)
            eventBus.on('pagechange', this.pageChangeEventHandler)
            this.pdf.eventBus = eventBus
        },
        pagesInitEventHandler(e) {
            this.setInitialPagination(e.source)
            this.updateScaleType()
            this.setInitialScale(e.source)
        },
        scaleChangeEventHandler(e) {
            this.scale.current = e.scale
            if (e.presetValue) this.scale.type = e.presetValue
        },
        pageChangeEventHandler(e) {
            this.pagination.current = e.pageNumber
        },
        createPdfLinkService() {
            this.pdf.linkService = new PDFJSViewer.PDFLinkService({
                eventBus: this.pdf.eventBus
            })
        },
        createPdfViewer() {
            this.pdf.viewer = new PDFJSViewer.PDFViewer({
                container: this.pdf.container,
                eventBus: this.pdf.eventBus,
                linkService: this.pdf.linkService
            })
        },
        renderDocument() {
            if (!this.src) return this.stopLoadingPlaceHolder()
            PDFJSLib.getDocument({
                url: this.src,
                httpHeaders: this.httpHeader,
                withCredentials: true
            })
                .then(pdf => {
                    this.pdf.linkService.setViewer(this.pdf.viewer)
                    this.pdf.linkService.setDocument(pdf, null)
                    this.pdf.viewer.setDocument(pdf)
                    this.stopLoadingPlaceHolder()
                })
                .catch(error => {
                    this.stopLoadingPlaceHolder()
                    this.handlePdfError(error)
                })
        },
        async createPrinterService() {
            this.pdf.printService = new PrintService({
                pdfDocument: this.pdf.viewer.pdfDocument,
                pagesOverview: await this.pdf.viewer.getPagesOverview()
            })
        },
        setInitialPagination(pagination) {
            this.pagination.current = pagination.currentPageNumber
            this.pagination.total = pagination.pagesCount
        },
        updateScaleType(scaleType) {
            if (scaleType && typeof scaleType === 'string') {
                this.pdf.viewer.currentScaleValue = scaleType
            } else {
                if (this.defaultScaleType && typeof this.defaultScaleType === 'string') {
                    this.pdf.viewer.currentScaleValue = this.defaultScaleType
                } else if (this.isSmallScreen()) {
                    this.pdf.viewer.currentScaleValue = 'page-width'
                } else {
                    this.pdf.viewer.currentScaleValue = 'page-fit'
                }
            }
        },
        setInitialScale(scale) {
            this.scale.current = scale.currentScale
            this.scale.default = scale.currentScale
            this.scale.type = scale.currentScaleValue
        },
        isSmallScreen() {
            return this.pdf.container.clientWidth <= 700
        },
        handlePdfError(error) {
            if (error.name === 'InvalidPDFException') {
                throw new Error('URL do documento inválida.')
            } else if (error.name === 'MissingPDFException') {
                throw new Error(error.message.replace('Missing PDF', 'Documento não encontrado em'))
            } else if (error.status === 401) {
                throw new Error('Não autorizado, verifique seu token de acesso.')
            }
        },
        changeScaleType() {
            if (this.scale.type === 'page-fit') {
                this.updateScaleType('page-width')
            } else {
                this.updateScaleType('page-fit')
            }
        },
        zoomIn() {
            this.pdf.viewer.currentScale = this.scale.current * 1.1
        },
        zoomOut() {
            if (this.scale.current > 0.2) {
                this.pdf.viewer.currentScale = this.scale.current / 1.1
            }
        },
        rotate() {
            this.pdf.viewer.pagesRotation = (this.pdf.viewer.pagesRotation + 90) % 360
        },
        download() {
            this.$emit('download')
        },
        async print() {
            this.startLoadingPrint()

            await this.createPrinterService()
            this.pdf.printService.prepareLayout()
            await this.pdf.printService.renderPages((currentPage, pageCount) => {
                this.printProgress = (currentPage / pageCount) * 100
            })
            window.print()
            this.pdf.printService.destroy()

            this.stopLoadingPrint()
        },
        cancelPrint() {
            this.pdf.printService.destroy()
            this.stopLoadingPrint()
        },
        startLoadingPlaceHolderIfNecessary() {
            this.loadingPlaceHolder = this.progressBar
        },
        stopLoadingPlaceHolder() {
            this.loadingPlaceHolder = false
        },
        startLoadingPrint() {
            this.isPrinting = true
        },
        stopLoadingPrint() {
            this.isPrinting = false
        }
    },
    props: {
        src: {
            type: String,
            default: ''
        },
        cssClass: {
            type: String,
            default: ''
        },
        height: {
            type: String,
            default: '100vh'
        },
        httpHeader: {
            type: Object,
            default: () => new Object()
        },
        progressBar: {
            type: Boolean,
            default: false
        },
        downloadButton: {
            type: Boolean,
            default: false
        },
        defaultScaleType: {
            type: String,
            default: ''
        },
        rotateButton: {
            type: Boolean,
            default: false
        },
        printButton: {
            type: Boolean,
            default: false
        }
    },
    computed: {
        customContainerClass() {
            let classObject = {}
            if (this.cssClass) {
                const classes = this.cssClass.split(' ')
                classes.forEach(clazz => (classObject[clazz] = true))
            }
            return classObject
        }
    },
    data: () => ({
        isPrinting: false,
        loadingPlaceHolder: false,
        pagination: {
            current: null,
            total: null
        },
        pdf: {
            container: null,
            eventBus: null,
            linkService: null,
            printService: null,
            viewer: {}
        },
        printProgress: 0,
        scale: {
            default: null,
            current: null,
            type: ''
        }
    })
}
</script>

<style src="pdfjs-dist/web/pdf_viewer.css"></style>

<style lang="stylus">
.az-pdf-container
    background-color #e4e4e4

    .Viewer
        width 100%
        position relative
        overflow scroll
        background-color #e4e4e4

        .page
            margin 15px auto
            border none
            border-image none

            .canvasWrapper
                -webkit-box-shadow 0 3px 6px -1px rgba(0, 0, 0, .2)
                box-shadow 0 3px 6px -1px rgba(0, 0, 0, .2)

@media print
    body[data-pdf-printing] #app
        display none

    body[data-pdf-printing] #print-container
        display block

        img
            display block
            width 100%
            height 100%

    body[data-pdf-printing] #print-container:first-child
        position relative
        top 0
        left 0
        width 1px
        height 1px
        overflow visible
        page-break-after always
        page-break-inside avoid

    body[data-pdf-printing] #print-container div
        page-break-after always
        page-break-inside avoid
</style>
