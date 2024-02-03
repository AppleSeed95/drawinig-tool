<template>
    <div>
        <div v-for="(item, idx) in Image" :key="idx" style="border:solid 1px #000000;">
            <ChildComponent 
                :backgroundImage="item.background"
                :drawingImage="item.drawingImage"
                :mode="mode"
                :color="color"
                :lineWidth="lineWidth"
            />
        </div>



    </div>
   
</template>


<script>


    import ChildComponent from './ChildComponent.vue'
    export default {
    data() {
        return {
            Image: [
            {
                    background: '/page1.jpg',
                    drawingImage: '/write1.png'
                },
                {
                    background: '/page2.jpg',
                    drawingImage: '/write2.png'
                },
            ],
            mode: "line",
            color: "#000000",
            lineWidth: 1,
            page: 0,
        }
    },
    components:{
        ChildComponent
    }
}
</script>
