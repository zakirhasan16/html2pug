<script setup>
/* eslint-disable */
import { computed, ref } from 'vue'

const defaultHTML = ref('')

const result = computed(() => {
  return parse()
})

const parse = () => {
  let string = defaultHTML.value

  string = string.replaceAll(/<([^>]*)>/gm, m => {
    let { inner } = new RegExp(/<(?<inner>[^>]*)>/gm).exec(m)?.groups || {}

    if(inner) {
      //! Tag Name'lerde TitleCase veya Camel Case olamaz, Kebap Case'e dönüştürmeliyiz.
      let { tagName } = new RegExp(/(?<tagName>[\w-]+)[ |\n]/m).exec(inner)?.groups || {}

      if(tagName) {
        if (tagName.match(/[A-Z][a-z]+/g)) {
          let newTagName = tagName.replaceAll(/[A-Z][a-z]+/g, m => `${ m.toLowerCase() }-`).slice(0, -1)

          m = m.replace(tagName, newTagName)
        }
      }
      // console.log(tagName.match(/[A-Z][a-z]+/g))
    }

    //! Attribute Camel Case kabul etmiyor. Camel Case'leri Kebap Case Dönüştürmeliyiz.
    if(m.match(/([\w:\-@]+)(?=(="))/gm)) {
      for (let attributeName of m.match(/([\w:\-@]+)(?=(="))/gm)) {
        if(attributeName.match(/[A-Z][a-z]+/g)) {
          let newAttributeName = attributeName.replaceAll(/[A-Z][a-z]+/g, m => `-${m.toLowerCase()}`)

          m = m.replace(attributeName, newAttributeName)
        }
      }
    }
    return m
  })

  //! Short Tagları Bulup kapatıyoruz.
  string = string.replaceAll(/<(?<inner>[^>]*)\/>/gm, m => {
    let { inner } = new RegExp(/<(?<inner>[^>]*)\/>/gm).exec(m)?.groups || {}

    if(inner) {
      let { tagName } = new RegExp(/(?<tagName>[\w-]+)[ |\n]/m).exec(inner)?.groups

      if(tagName) return `${m}</${tagName}>`
    }

    return m
  })

  //! Template ile çerçevelenmiş bloklar sorun yaratıyor onları şimdilik bambaşka bir şekle çeviriyoruz sonra çıktıyı template olarak vereceğiz.
  string = string.replaceAll('<template', '<zzzemplate').replaceAll('</template>', '</zzzemplate>')


  //! Elimizdeki string'den html'imizi oluşturuyoruz.
  const doc = new DOMParser().parseFromString(string, 'text/html')
  const elements = [...doc.body.children]
  let result = ''

  console.log(doc.body.children)

  //! Bu Fonksiyon bir child elementin derinliğini bulmamıza yarıyor. Bu derinliği tab ayarlamak için kullanıyoruz.
  const getNodeDepth = node => {
    let [depth, parent] = [0, node.parentNode]
    while(parent !== doc.body) {
      parent = parent.parentNode
      depth++
    }
    return depth
  }

  //! Derinlik kadar Tab ekleyen fonksiyonumuz
  const addDepthSpaces = d => Array.from({length: d}).reduce((acc) => `${acc}\t`, '')

  //! Verdiğimiz node'u ve child node'larını result'a basan recursive fonksiyonumuz.
  const childNodesToPug = (node) => {
    let d = getNodeDepth(node)
    //! Text ise
    if(node.nodeType === 3) {
      let text = node.nodeValue.trim()
      if(text.startsWith('\n')) {
        text = `\n${addDepthSpaces(d)}${text.replaceAll('\n','')}`
      }
      result += text
    }
    //! Comment ise
    if(node.nodeType === 8) {
      result += `\n${addDepthSpaces(d)}// ${node.nodeValue}`
    }
    //! Element ise
    if(node.nodeType === 1) {
      result += `\n${addDepthSpaces(d)}${elToPug(node)} `

      if(node.childNodes.length) {
        const childNodes = [...node.childNodes]

        childNodes.forEach(n => {
          childNodesToPug(n)
        })
      }
    }
  }

  //! Bir HTML elemanı alıp Pug yazımına çeviren fonksiyonumuz.
  const elToPug = (el) => {
    let tagName = el.tagName.toLowerCase()
    let d = getNodeDepth(el)

    let excludedAttributes = ['class', 'id']
    let attributes = [...el.attributes].filter(attr => !excludedAttributes.includes(attr.name)).map((attr) => `${attr.name}${attr.value ? `="${attr.value}"` : ''}`)
    if(attributes.length >= 4) {
      attributes = attributes.join(`\n${addDepthSpaces(d+1)}`)
    } else {
      let attributesText = attributes.join(' ')

      if(attributesText.length >= 250) {
        attributes = attributes.join(`\n${addDepthSpaces(d+1)}`)
      } else {
        attributes = attributesText
      }
    }
    attributes = attributes.trim()

    const classes = [...el.classList].join('.')

    if(classes && tagName === 'div') tagName = ''

    if(tagName === 'zzzemplate') tagName = 'template'

    return `${tagName}${el.id ? `#${el.id}` : ''}${classes ? `.${classes}` : ''}${attributes ? `(${attributes.match('\n') ? `\n${addDepthSpaces(d+1)}${attributes}\n${addDepthSpaces(d)}` : attributes})` : ''}`
  }

  elements.forEach(el => {
    result += elToPug(el)

    if(el.childNodes.length) {
      let childNodes = [...el.childNodes]
      childNodes.forEach(n => childNodesToPug(n))
    }
  })

  return result
}
</script>

<template>
  <v-container fluid class="fill-height d-flex w-100">
    <div class="fill-height d-flex flex-column w-100">
      <h6>HTML</h6>
      <textarea class="w-100 fill-height border" v-model="defaultHTML" />
    </div>
    <div class="fill-height d-flex flex-column w-100">
      <h6>PUG Response</h6>
      <textarea class="w-100 fill-height border" :value="result" />
    </div>
  </v-container>
</template>