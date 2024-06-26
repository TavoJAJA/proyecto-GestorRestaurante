<script setup>
// importaciones de vue
import { computed, onMounted, ref } from 'vue';
import { useRoute, useRouter } from 'vue-router';

// importaciones de firebase
import { storage } from '../../firebaseConfig';

// importando el store de productos
import { useProductosStore } from '../../stores/productos';

// importando funciones de utilidades
import { encontrarProducto, isEmptyObject, printf, uploadFile } from '../../utilidades';
import { deleteObject, ref as firebaseRef } from 'firebase/storage';
import { useMensajesStore } from '../../stores/mensajes';

// inicializando los stores
const productosStore = useProductosStore()
const mensajesStore = useMensajesStore()

// objetos para utilizar la ruta actual y el router para redirigir a paginas, respectivamente
const route = useRoute()
const router = useRouter()

// id del producto del que se mostrara informacion
const idProducto = route.params.id;

// lista de productos
const tiposProductos = productosStore.listarTipoProductos

// variables reactivas que muestran la informacion del producto
const nombre = ref(null)
const descripcion = ref(null)
const precio = ref(null)
const disponibilidad = ref(null)
const tipo = ref(null)
const foto = ref(null)  // url de la imagen en Storage

// copias de las variables reactivas del producto, se usaran para el proceso de actualizacion del registro en firestore
const nombreCopia = ref(null)
const descripcionCopia = ref(null)
const precioCopia = ref(null)
const disponibilidadCopia = ref(null)
const tipoCopia = ref(null)
const fotoCopia = ref(null)  // url de la imagen en Storage

const previewImage = ref(null)  // variable reactiva para mostrar la imagen seleccionada
const tipoProductoOption = ref(null) // para seleccionar automaticamente en el input select el tipo del producto
const fotoActualizada = ref(false) // bandera para saber si el administrador cambio la imagen del producto
const fotoOriginal = ref(null)  // valor de la foto original, se usa en caso de actualizar la imagen existente por otra nueva
const bloquearBoton = ref(false)
const claseValidacion = ref('') // clases de bootstrap para mostrar mensajes de validacion

// recuperando objeto con la informacion del producto, esta informacion debe cargarse antes de que se renderice el HTML del componente
const producto = computed(() => {
    const productoInfo = encontrarProducto(productosStore.productos, idProducto)

    // variables reactivas de informacion del producto
    nombreCopia.value = nombre.value = productoInfo?.nombre
    descripcionCopia.value = descripcion.value = productoInfo?.desc
    precioCopia.value = precio.value = productoInfo?.precio
    disponibilidadCopia.value = disponibilidad.value = productoInfo?.disponible
    tipoCopia.value = tipo.value = productoInfo?.tipo
    fotoCopia.value = foto.value = productoInfo?.foto

    // otroas variables que se usaran
    tipoProductoOption.value = productoInfo?.tipo
    fotoOriginal.value = productoInfo?.foto
    previewImage.value = productoInfo?.foto

    return productoInfo;
})

// funcion para poder mostrar la imagen en el div
const handleFileUpload = e => {
    const file = e.target.files[0]

    if (!file.type.startsWith('image/')) {
        // Mostrar un mensaje de error indicando que solo se permiten imágenes
        mensajesStore.crearMensaje({
            titulo: 'Archivo no válido',
            texto: 'Solo puede seleccionar archivos de imágenes válidos',
            color: 'warning',
            id: 'noCargaImagenEditar',
            autoEliminar: true
        })
        previewImage.value = null
        return;
    }

    foto.value = file

    if (file) {
        const reader = new FileReader();
        reader.onload = (e) => {
            previewImage.value = e.target.result;
            fotoActualizada.value = true
        };
        reader.readAsDataURL(file);
    }
}


// funcion para actualizar los datos del producto
const actualizarProducto = async () => {
    // se hace validacion del formulario, para que no senvien campos en blanco
    const validacionCorrecta = validarFormulario()


    if (!validacionCorrecta) {
        claseValidacion.value = 'was-validated'
        return;
    } else {
        claseValidacion.value = ''
    }

    let fotoActualizarURL = foto.value
    bloquearBoton.value = true; // desactiva el boton de actualizacion durante el proceso

    try {
        // Primer paso consiste en comprobar si se ha cambiado la foto, y si es el caso, eliminarla y subir la nueva
        if (fotoActualizada.value) {
            // crear referencia a la foto existene del producto en base a su URI en Storage para eliminarla
            const refFotoVieja = firebaseRef(storage, fotoOriginal.value)

            // se elimina la foto existente
            await deleteObject(refFotoVieja)

            // se sube la foto nueva seleccionada en la carpeta 'productos' en Storage y regresa su URL
            fotoActualizarURL = await uploadFile(foto.value, 'productos')
        }

        // los campos que no se actualizan en el formulario no se actulizaran tampoco en firestore
        // este objeto es temporal, se debera filtrar los campos que no se actualizaran con la funcion 'generateUpdateObject'
        const productoActualizadoTemp = {
            nombre: nombre.value === nombreCopia.value ? undefined : nombre.value,
            desc: descripcion.value === descripcionCopia.value ? undefined : descripcion.value,
            precio: precio.value === precioCopia.value ? undefined : precio.value,
            disponible: disponibilidad.value === disponibilidadCopia.value ? undefined : disponibilidad.value,
            tipo: tipo.value === tipoCopia.value ? undefined : tipo.value,
            foto: fotoActualizarURL === fotoCopia.value ? undefined : fotoActualizarURL
        }

        // el objeto resultante deberia tener solos los campos cuya informacion se actualizara
        const productoActualizado = generateUpdateObject(productoActualizadoTemp)

        // se comprueba que el objeto a actualizar no sea vacio, para no hacer nada mas en la funcion
        if (isEmptyObject(productoActualizado)) {
            mensajesStore.crearMensaje({
                titulo: 'Nada que actualizar',
                texto: 'No ocurrión actualización porque los datos a actualizar son exactamente los mismos que existen en la base de datos',
                color: 'secondary',
                id: 'nadaQueActualizarProducto',
                autoEliminar: true
            })

            return
        } else {
            // se manda el objeto a actualizar al store de pinia donde se llama el proceso a firestore
            await productosStore.actualizarProducto(productoActualizado, idProducto)

            mensajesStore.crearMensaje({
                titulo: 'Producto actualizado',
                texto: 'La información del producto se actualizó exitosamente',
                color: 'success',
                id: 'productoActualizado',
                autoEliminar: true
            })

            // se reasignan los valores de las copias con los valores actualizados
            nombreCopia.value = nombre.value
            descripcionCopia.value = descripcion.value
            precioCopia.value = precio.value
            disponibilidadCopia.value = disponibilidad.value
            tipoCopia.value = tipo.value
            fotoCopia.value = foto.value
        }

    } catch (error) {
        mensajesStore.crearError('errorActualizarProducto', 'No se pudo actualizar el producto')
        console.log(error)
    } finally {
        bloquearBoton.value = false
    }
}

// Función para generar el objeto de actualización dinámico, para no actualizar todo el objeto innecesariamente si solo se modifican algunos campos
const generateUpdateObject = administradorActualizaciones => {
    const updateObject = {};
    for (const [campo, valor] of Object.entries(administradorActualizaciones)) {
        if (valor !== undefined) {
            updateObject[campo] = valor;
        }
    }
    return updateObject;
}

// validar el formulario
const validarFormulario = () => {
    return nombre.value && descripcion.value && precio.value
}


</script>

<template>
<<<<<<< HEAD
<br>
<h1>Editar informacion del producto:</h1>
<br>
<div class="container text-center">
    
  <div class="row">
    <div class="col">
                <!-- Muestra la imagen seleccionada -->
        <div class="prevImagen">
            <div class="border border-default">
                <div class="imagePreviewWrapper" :style="{ 'background-image': `url(${foto})` }">
                </div>
            </div>
            <div class="invalid-feedback">
                Por favor elija una imagen.
            </div>
        </div>
                <!-- Seleccionar imagen para -->
        <div class="prevImagen">
            <label for="seleccionarImagen" required class="form-label">Seleccione una imagen para el
                producto:</label>
            <input class="form-control" type="file" id="seleccionarImagen" @change="handleFileUpload">
            <div class="invalid-feedback">
                Por favor elija una imagen.
            </div>
        </div>

    </div>
    <div class="col">
        <label for="basic-url" class="form-label"></label>
            <div class="input-group mb-3">
                <span class="input-group-text" id="basic-addon3">Nombre:</span>
                <input type="text" class="form-control" id="txtCambiarNombreProducto" required placeholder="Ingrese el nombre" aria-describedby="basic-addon3">
                <div class="invalid-feedback">
                    Por favor ingrese el nombre.
                </div>

            </div>

            <div class="input-group" required id="txtCambiarDescripcionProducto" >
                <span class="input-group-text">Descripcion:</span>
                <textarea class="form-control" aria-label="With textarea"></textarea>
                <div class="invalid-feedback">
                    Por favor ingrese la descripcion.
                </div>
            </div>

            <label for="basic-url" class="form-label"></label>
            <div class="input-group mb-3">
                <span class="input-group-text" id="basic-addon3">Precio $</span>
                <input type="number" class="form-control" min="0" id="txtCambiarPrecioProducto" required placeholder="Ingrese el precio" aria-describedby="basic-addon3">
                <div class="invalid-feedback">
                    Por favor ingrese el precio.
                </div>
            </div>

            <select class="form-select" aria-label="Default select example" required id="elegirTipoProducto">
                 <option selected disabled>Tipo de producto:</option>
                 <option value="1">Postre</option>
                 <option value="2">Bebida Caliente</option>
                 <option value="3">Bebida Fria</option>
                 <div class="invalid-feedback">
                    Por favor elija una opcion.
                </div>
            </select>
            <br>
            <div class="container text-center">
=======
    <br>
    <h1>Editar informacion</h1>
    <br>
    <div class="container text-center" v-if="producto">

        <form :class="claseValidacion" @submit.prevent>
>>>>>>> c8f73713887e388061b1a48800f2a2837709d226
            <div class="row">
                <div class="col">
                    <!-- Muestra la imagen seleccionada -->
                    <div class="prevImagen">
                        <div class="border border-default">
                            <div class="imagePreviewWrapper" :style="{ 'background-image': `url(${previewImage})` }">
                            </div>
                        </div>
                        <div class="invalid-feedback">
                            Por favor elija una imagen.
                        </div>
                    </div>
                    <!-- Seleccionar imagen para -->
                    <div class="prevImagen">
                        <label for="seleccionarImagen"  class="form-label">Seleccione una imagen para el
                            producto:</label>
                        <input class="form-control" type="file" id="seleccionarImagen" @change="handleFileUpload"
                            accept="image/*">
                    </div>

                </div>
                <div class="col">
                    <div class="input-group mb-3">
                        <span class="input-group-text" id="basic-addon3">Nombre:</span>
                        <input type="text" class="form-control" id="txtCambiarNombreProducto" required
                            placeholder="Ingrese el nombre" aria-describedby="basic-addon3" v-model.trim="nombre">
                        <div class="invalid-feedback">
                            Por favor ingrese el nombre.
                        </div>

                    </div>

                    <div class="input-group mb-3" required id="txtCambiarDescripcionProducto">
                        <span class="input-group-text">Descripcion:</span>
                        <textarea class="form-control" aria-label="With textarea" v-model.trim="descripcion"
                            required></textarea>
                        <div class="invalid-feedback">
                            Por favor ingrese la descripcion.
                        </div>
                    </div>

                    <div class="input-group mb-3">
                        <span class="input-group-text" id="basic-addon3">Precio ($):</span>
                        <input type="number" class="form-control" id="txtCambiarPrecioProducto" required
                            placeholder="Ingrese el precio" min="0" aria-describedby="basic-addon3" v-model.number="precio">
                        <div class="invalid-feedback">
                            Por favor ingrese el precio.
                        </div>
                    </div>
                    
                    <div class="input-group mb-3">
                        <span class="input-group-text" id="basic-addon3">Disponibilidad:</span>
                        <select class="form-select" v-model="disponibilidad" required>
                            <option value="" disabled>Elija una opción</option>
                            <option :value="true" :selected="disponibilidad">Disponible</option>
                            <option :value="false" :selected="!disponibilidad">No disponible</option>
                        </select>
    
                        <div class="invalid-feedback">
                            Por favor ingrese la disponibilidad
                        </div>
                    </div>

                    <div class="input-group mb-3">
                        <span class="input-group-text" id="basic-addon3">Tipo del producto:</span>
                        <select class="form-select" aria-label="Default select example" required id="elegirTipoProducto"
                            v-model.trim="tipo">
                            <option selected disabled value="">Seleccione una opción</option>
                            <template v-for="tipo in tiposProductos" :key="tipo.id">
                                <option :value="tipo.nombre" :selected="tipoProductoOption === tipo.nombre">{{
                                    tipo.nombre
                                }}</option>
                            </template>
                            <div class="invalid-feedback">
                                Por favor elija una opcion.
                            </div>
                        </select>
                    </div>
                    <br>
                    <div class="container text-center">
                        <div class="row">
                            <div class="col">
                                <!-- Muestra el div hasta que se hallan traido los datos de la base de datos -->
                                <button class="btn btn-outline-success" @click="actualizarProducto"
                                    :disabled="bloquearBoton">Actualizar
                                    producto</button>
                            </div>
                        </div>
                    </div>


                </div>
            </div>
        </form>
    </div>

    <!-- Reemplazar lo que esta dentro de este div con un loading spinner  -->
    <div v-else>
        <div class="spinner-border" role="status">
            <span class="visually-hidden">Cargando...</span>
        </div>
    </div>

    <div class="mt-4 d-flex justify-content-center ">
        <!-- Boton que regresa al menu -->
        <button type="button" class="btn btn-lg btn-outline-info" @click="router.push('/menu')">Ir al menú</button>
    </div>
</template>
<style scoped>
.imagePreviewWrapper {
    width: 250px;
    height: 250px;
    display: block;
    cursor: pointer;
    margin: 0 auto 30px;
    background-size: cover;
    background-position: center center;
}
</style>
